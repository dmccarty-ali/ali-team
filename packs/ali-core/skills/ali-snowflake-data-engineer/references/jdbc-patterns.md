# ali-snowflake-data-engineer - Java JDBC Integration Reference

## Connection Configuration

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.util.Properties;

public class SnowflakeConfig {
    private final Properties connectionProps;

    public SnowflakeConfig(String account, String database, String schema, String warehouse) {
        this.connectionProps = new Properties();
        connectionProps.put("account", account);
        connectionProps.put("db", database);
        connectionProps.put("schema", schema);
        connectionProps.put("warehouse", warehouse);
    }

    // Password authentication
    public Connection getConnection(String user, String password) throws SQLException {
        connectionProps.put("user", user);
        connectionProps.put("password", password);
        return DriverManager.getConnection(
            "jdbc:snowflake://" + connectionProps.getProperty("account") + ".snowflakecomputing.com",
            connectionProps
        );
    }

    // RSA key pair authentication (recommended for automation)
    public Connection getConnectionWithKey(String user, String privateKeyPath) throws Exception {
        connectionProps.put("user", user);
        connectionProps.put("authenticator", "SNOWFLAKE_JWT");

        // Load PKCS8 private key
        String privateKeyContent = Files.readString(Path.of(privateKeyPath))
            .replace("-----BEGIN PRIVATE KEY-----", "")
            .replace("-----END PRIVATE KEY-----", "")
            .replaceAll("\\s", "");

        byte[] keyBytes = Base64.getDecoder().decode(privateKeyContent);
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        PrivateKey privateKey = keyFactory.generatePrivate(keySpec);

        connectionProps.put("privateKey", privateKey);
        return DriverManager.getConnection(
            "jdbc:snowflake://" + connectionProps.getProperty("account") + ".snowflakecomputing.com",
            connectionProps
        );
    }
}
```

## Query Tag for Monitoring

```java
// Set query tag for tracking in QUERY_HISTORY
public void setQueryTag(Connection conn, String tag) throws SQLException {
    try (Statement stmt = conn.createStatement()) {
        stmt.execute("ALTER SESSION SET QUERY_TAG = '" + tag + "'");
    }
}

// Usage: Track which batch/parser generated each query
setQueryTag(conn, "batch_id=" + batchId + ",parser=edi_835,stage=merge");

// Query tagged operations later
// SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
// WHERE QUERY_TAG LIKE '%batch_id=abc123%';
```

## PUT Operations from Java

```java
public class SnowflakeStage {
    private final Connection connection;
    private final String stageName;

    /**
     * Upload file to internal stage using PUT command.
     * Returns metadata about the uploaded file.
     */
    public PutResult putFile(Path localPath, String stagePath) throws SQLException {
        // PUT requires forward slashes and file:// prefix
        String sql = String.format(
            "PUT 'file://%s' '%s/%s' AUTO_COMPRESS=FALSE OVERWRITE=TRUE",
            localPath.toString().replace("\\", "/"),
            stageName,
            stagePath
        );

        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {

            if (rs.next()) {
                return new PutResult(
                    rs.getString("source"),
                    rs.getString("target"),
                    rs.getString("status"),      // UPLOADED, SKIPPED
                    rs.getLong("source_size"),
                    rs.getLong("target_size"),
                    rs.getString("message")
                );
            }
            throw new SQLException("No result from PUT operation");
        }
    }

    /**
     * Upload multiple files matching pattern.
     */
    public List<PutResult> putFiles(Path directory, String pattern, String stagePath)
            throws SQLException, IOException {
        List<PutResult> results = new ArrayList<>();

        try (DirectoryStream<Path> stream = Files.newDirectoryStream(directory, pattern)) {
            for (Path file : stream) {
                results.add(putFile(file, stagePath));
            }
        }
        return results;
    }
}

// PutResult record
public record PutResult(
    String source,
    String target,
    String status,
    long sourceSize,
    long targetSize,
    String message
) {
    public boolean isSuccess() {
        return "UPLOADED".equals(status);
    }
}
```

## Dynamic Warehouse Parallelism

```python
# Map warehouse sizes to compute nodes
WAREHOUSE_NODE_COUNT = {
    'X-SMALL': 1,
    'SMALL': 2,
    'MEDIUM': 4,
    'LARGE': 8,
    'X-LARGE': 16,
    '2X-LARGE': 32,
    '3X-LARGE': 64,
    '4X-LARGE': 128,
}

def get_warehouse_size(conn) -> str:
    """Get current warehouse size."""
    cursor = conn.cursor()
    cursor.execute("SELECT CURRENT_WAREHOUSE()")
    warehouse_name = cursor.fetchone()[0]

    cursor.execute(f"SHOW WAREHOUSES LIKE '{warehouse_name}'")
    row = cursor.fetchone()
    # Size is typically in column index 3
    return row[3].upper()  # 'X-Small' -> 'X-SMALL'

def get_warehouse_thread_count(conn, threads_per_node: int = 4) -> int:
    """Calculate thread count based on warehouse size."""
    size = get_warehouse_size(conn)
    nodes = WAREHOUSE_NODE_COUNT.get(size, 1)
    return nodes * threads_per_node

def get_copy_parallelism(conn) -> int:
    """Get optimal COPY parallelism (more conservative than threads)."""
    # COPY operations are I/O heavy, use fewer parallel operations
    COPY_PARALLELISM = {
        'X-SMALL': 2,
        'SMALL': 3,
        'MEDIUM': 5,
        'LARGE': 8,
        'X-LARGE': 12,
        '2X-LARGE': 16,
        '3X-LARGE': 24,
        '4X-LARGE': 32,
    }
    size = get_warehouse_size(conn)
    return COPY_PARALLELISM.get(size, 2)
```

## Usage in Pipeline

```python
# Adjust batch sizes based on warehouse
def process_files(conn, files: list[Path]):
    parallelism = get_copy_parallelism(conn)

    # Split files into batches for parallel COPY
    batches = [files[i:i + parallelism] for i in range(0, len(files), parallelism)]

    for batch in batches:
        # Execute COPY operations for batch
        copy_files_to_table(conn, batch)

# Adjust thread pool for parallel processing
def run_parsers(conn, parser_configs: list):
    thread_count = get_warehouse_thread_count(conn)

    with ThreadPoolExecutor(max_workers=thread_count) as executor:
        futures = [executor.submit(run_parser, cfg) for cfg in parser_configs]
        for future in as_completed(futures):
            result = future.result()
```
