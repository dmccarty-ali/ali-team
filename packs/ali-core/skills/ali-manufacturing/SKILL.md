---
name: ali-manufacturing
description: |
  Manufacturing domain knowledge covering data models, KPIs, ERP structures, and
  analytics patterns for discrete and process manufacturing. Use when:

  PLANNING: Designing analytics solutions for manufacturing, planning data models for
  production/quality/supply chain, architecting ERP integration patterns, evaluating
  IoT telemetry pipelines, designing dealer network analytics

  IMPLEMENTATION: Building manufacturing data marts or medallion layers, implementing
  BOM explosion logic, writing OEE/production KPI calculations, creating supply chain
  scorecards, modeling ERP source tables (SAP PP/MM/QM, IFS, BAAN), building
  predictive maintenance pipelines

  GUIDANCE: Asking about manufacturing domain concepts (BOM, MRP, OEE, MES), best
  practices for ERP data integration, KPI definitions, data modeling patterns for
  multi-plant environments, IoT/telemetry architecture

  REVIEW: Reviewing manufacturing analytics designs, validating KPI definitions,
  checking BOM data model correctness, auditing ERP integration patterns, evaluating
  supply chain scorecard completeness

  DO NOT use for:
  - Generic data warehouse design without manufacturing context (use ali-data-architecture)
  - SAP Basis/infrastructure (use ali-sap-mdg for MDM patterns)
  - General logistics/transportation without manufacturing origin
---

# Manufacturing Domain Knowledge

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing data models for production, quality, supply chain, or warehouse analytics
- Planning medallion architecture layers for manufacturing source data
- Architecting ERP integration for SAP PP/MM/QM, IFS, BAAN, or Infor LN
- Evaluating IoT telemetry pipelines for connected equipment or predictive maintenance
- Designing dealer network or fleet analytics

**Implementation:**
- Building star schemas or dimensional models for manufacturing
- Implementing BOM explosion, where-used, or cost rollup logic
- Writing OEE, yield, scrap rate, or cycle time calculations
- Creating supply chain scorecards (OTD, fill rate, inventory turns)
- Modeling ERP source tables and change data capture patterns
- Building time-series pipelines for machine telemetry

**Guidance/Best Practices:**
- Asking about manufacturing KPI definitions (OEE formula, COPQ, DPMO)
- Asking about BOM types and data model patterns
- Asking how MRP/MPS works and what data it needs
- Asking about ERP module data structures (SAP PP vs IFS vs BAAN)
- Asking about quality management data flows
- Asking about lift truck / material handling industry specifics

**Review/Validation:**
- Reviewing manufacturing analytics designs for domain correctness
- Validating KPI formulas against industry standards
- Checking BOM and routing data model completeness
- Auditing ERP integration patterns for terminology or granularity issues
- Evaluating supply chain or dealer scorecard completeness

---

## Key Principles

- **OEE = Availability x Performance x Quality**: Never simplify this to a single metric or use a proxy
- **BOM is hierarchical**: Always account for multi-level explosion; phantom assemblies and circular references must be handled
- **ERP terminology diverges**: SAP calls it a "Production Order"; IFS calls it a "Work Order"; BAAN calls it a "Production Order" but with different status codes -- always clarify source
- **Unit of measure is a first-class concern**: Manufacturing data has base UOM, order UOM, and reporting UOM -- conversions must be explicit
- **Factory calendar != fiscal calendar**: Production analytics use factory/shift calendars; financial analytics use fiscal calendars -- joins require explicit calendar bridging
- **Quality is traceability**: Every defect and warranty claim must trace back to a production lot, BOM revision, and work center
- **Inventory has stock types**: Unrestricted, blocked, quality inspection, transit, and consignment are not interchangeable -- never aggregate across types without intent
- **Dealer networks are hierarchical**: Region > District > Dealer; territory analytics require hierarchy bridging
- **IoT data is high-volume and late-arriving**: Time-series pipelines need partitioning, compaction, and late-arrival handling

---

## 1. Manufacturing Data Domains

### Master Data

| Entity | Purpose | Key Attributes | ERP Examples |
|--------|---------|----------------|--------------|
| Material / Product Master | Describes every item bought, made, or sold | Material #, description, material type, base UOM, plant assignments, MRP type | SAP MARA/MARC, IFS Part, BAAN Item |
| Bill of Materials (BOM) | Parent-child assembly structure | BOM header, component item, qty per, UOM, validity dates, alternative BOM | SAP CS01/STKO/STPO, IFS Part Structure |
| Routing / Operation | Process steps and work centers for production | Routing #, operation, work center, standard times (setup, machine, labor) | SAP CA01/PLPO, IFS Manufacturing Routing |
| Work Center | Machine or work cell where operations run | Work center, capacity category, available capacity, cost center | SAP CR01, IFS Work Center |
| Production Order / Work Order | Authorization to produce a quantity by a date | Order #, material, plant, qty, dates, status, BOM/routing snapshot | SAP CO01/AUFK, IFS Work Order, BAAN Production Order |
| Customer | Who buys finished goods or dealer | Customer #, name, region, territory, account type | SAP KNA1, IFS Customer |
| Vendor / Supplier | Who supplies raw materials or components | Vendor #, name, country, payment terms, evaluation score | SAP LFA1, IFS Supplier |
| Plant / Location | Manufacturing site or distribution center | Plant code, name, country, company code, calendar | SAP T001W, IFS Site |

### Transactional Data

| Entity | Purpose | Key Attributes |
|--------|---------|----------------|
| Goods Movement | Inventory movements (GR, GI, transfer) | Movement type, material, plant, qty, posting date, reference document |
| Production Confirmation | Actual qty produced, scrap, time per operation | Order #, operation, confirmed qty, scrap qty, actual times |
| Quality Inspection / Lot | Results of in-process or receiving inspection | Inspection lot, material, batch, usage decision, defect qty |
| Quality Notification | Defect or non-conformance record | Notification #, type, material, defect codes, corrective actions |
| Delivery (Outbound/Inbound) | Shipment of goods to customer or from supplier | Delivery #, material, qty, planned/actual dates, carrier |
| Purchase Order / Line | Authorization to buy from supplier | PO #, supplier, material, qty, price, delivery date, GR status |
| Sales Order / Line | Customer order for finished goods | SO #, customer, material, qty, requested date, confirmed date |

---

## 2. Bill of Materials

### BOM Types

| BOM Type | Description | Use Case |
|----------|-------------|---------|
| Single-Level BOM | Parent with direct children only | Simple assemblies, purchased sub-assemblies |
| Multi-Level BOM | Full explosion to raw material level | Complete cost rollup, MRP explosion |
| Variant / Configurable BOM | One BOM with configuration options | Product families (e.g., lift truck model lines) |
| Engineering BOM (EBOM) | Design structure from R&D | Change management, new product introduction |
| Manufacturing BOM (MBOM) | How item is actually built | Production orders, shop floor instructions |
| Service BOM | Parts needed for repair/service | Dealer service, field technician parts ordering |

### BOM Data Model Pattern (Star Schema)

```
DIM_BOM_HEADER
  bom_id (PK)
  parent_material_id (FK to DIM_MATERIAL)
  bom_type (EBOM / MBOM / SBOM)
  alternative_bom
  valid_from_date
  valid_to_date
  revision_level
  created_by
  created_date

FACT_BOM_COMPONENT (one row per parent-child relationship)
  bom_component_id (PK)
  bom_id (FK to DIM_BOM_HEADER)
  parent_material_id (FK to DIM_MATERIAL)
  component_material_id (FK to DIM_MATERIAL)
  level_number           -- 1 = direct child, 2 = grandchild, etc.
  item_number            -- sequence within BOM
  component_qty
  component_uom_id (FK to DIM_UOM)
  is_phantom_assembly    -- phantom = no stock, exploded through
  scrap_factor_pct       -- planned scrap percentage
  valid_from_date
  valid_to_date
```

### Multi-Level BOM Explosion

For analytics, pre-explode the BOM into a flattened structure using a recursive CTE or ETL job:

```sql
-- Recursive CTE for full BOM explosion
WITH RECURSIVE bom_explosion AS (
    -- Anchor: direct components (level 1)
    SELECT
        parent_material_id      AS top_parent_id,
        component_material_id,
        component_qty           AS extended_qty,
        1                       AS level_number,
        ARRAY[parent_material_id, component_material_id] AS path
    FROM fact_bom_component
    WHERE level_number = 1

    UNION ALL

    -- Recursive: explode children
    SELECT
        be.top_parent_id,
        bc.component_material_id,
        be.extended_qty * bc.component_qty AS extended_qty,
        be.level_number + 1,
        be.path || bc.component_material_id
    FROM bom_explosion be
    JOIN fact_bom_component bc ON be.component_material_id = bc.parent_material_id
    WHERE bc.component_material_id != ALL(be.path)  -- Circular reference guard
)
SELECT * FROM bom_explosion;
```

### Where-Used Analysis

"Where is this component used?" -- essential for engineering change impact and substitution analysis:

```sql
-- Find all parents that use component X
SELECT
    fbc.parent_material_id,
    dm.material_description,
    fbc.component_qty,
    fbc.component_uom_id
FROM fact_bom_component fbc
JOIN dim_material dm ON fbc.parent_material_id = dm.material_id
WHERE fbc.component_material_id = :component_material_id
  AND CURRENT_DATE BETWEEN fbc.valid_from_date AND fbc.valid_to_date;
```

### Key Considerations

- **Revision tracking**: Always include valid_from/valid_to dates on BOM components; production orders snapshot the BOM at creation time
- **Cost rollup**: Extended cost = component_qty x component_unit_cost; roll up from leaves to root through multi-level BOM
- **Phantom assemblies**: Phantoms are not stocked; MRP explodes through them to their components
- **Alternate BOMs**: SAP supports multiple alternative BOMs for same parent (different production methods)

---

## 3. Production Planning and Execution

### Planning Hierarchy

| Level | Process | Horizon | Output |
|-------|---------|---------|--------|
| Sales & Operations Planning (S&OP) | Demand vs capacity alignment | 12-18 months | Consensus demand plan |
| Master Production Schedule (MPS) | Planned finished goods production | 4-12 weeks | MPS quantities per item per period |
| Material Requirements Planning (MRP) | Component and material requirements | 2-8 weeks | Planned orders (production + purchase) |
| Capacity Requirements Planning (CRP) | Work center load vs capacity | 2-8 weeks | Capacity load reports |
| Production Execution (MES/ERP) | Shop floor confirmation | Real-time | Goods movements, confirmations |

### Production Order Lifecycle

```
Created (CRTD)
    --> Released (REL)    [BOM/routing exploded, materials reserved]
        --> Partially Confirmed (PCNF)  [some operations confirmed]
            --> Confirmed (CNF)         [all operations done, qty declared]
                --> Delivered (DLV)     [finished goods goods receipt posted]
                    --> Technically Complete (TECO)
                        --> Closed (CLSD)  [cost settlement complete]
```

### Production KPI Definitions

| KPI | Formula | Notes |
|-----|---------|-------|
| OEE | Availability x Performance x Quality | Industry standard; never proxy with a single ratio |
| Availability | (Planned Time - Downtime) / Planned Time | Planned time = shift time minus planned stops |
| Performance | (Ideal Cycle Time x Actual Output) / Run Time | Also: (Actual Output / Theoretical Output) |
| Quality Rate | Good Units / Total Units Started | Good = conforming, accepted units |
| Throughput | Good Units / Time Period | Units per hour, per shift, per day |
| Cycle Time | Total Production Time / Units Produced | Actual time per unit |
| Yield | Good Output / Input Material Qty | Often expressed as % |
| Scrap Rate | Scrap Qty / Total Started Qty | Complement of yield |
| Schedule Attainment | Actual Production / Scheduled Production | Adherence to MPS |

### OEE Data Model

```
FACT_OEE_DAILY (grain: work center + shift + date)
  oee_key (PK)
  work_center_id (FK)
  plant_id (FK)
  date_key (FK)
  shift_key (FK)
  planned_production_minutes
  unplanned_downtime_minutes
  planned_downtime_minutes
  actual_output_qty
  theoretical_output_qty
  good_output_qty
  scrap_qty
  -- Derived
  availability_pct
  performance_pct
  quality_pct
  oee_pct
```

---

## 4. Quality Management

### Quality Data Flows

```
Incoming Material  -->  Inspection Lot (Type 01)  -->  Usage Decision (Accept/Reject/Rework)
Production Yield   -->  Inspection Lot (Type 03)  -->  Usage Decision
Customer Return    -->  Quality Notification  -->  Root Cause Analysis  -->  Corrective Action
Warranty Claim     -->  Quality Notification  -->  Field Return / Failure Analysis
```

### Quality Notification Structure

```
FACT_QUALITY_NOTIFICATION
  notification_id (PK)
  notification_type      -- (B2 = customer complaint, F2 = internal, Q1 = supplier)
  material_id (FK)
  plant_id (FK)
  batch_id
  production_order_id    -- Traceability back to production
  customer_id (FK)       -- If customer complaint
  vendor_id (FK)         -- If supplier quality issue
  created_date_key (FK)
  defect_qty
  defect_code_id (FK)    -- DIM_DEFECT_CODE
  defect_location_id (FK)
  cause_code_id (FK)
  corrective_action_text
  status                 -- (OSNO, NOPR, NOPT, CLSD)
  days_open
  cost_of_quality_usd
```

### Quality KPI Definitions

| KPI | Formula | Notes |
|-----|---------|-------|
| First Pass Yield (FPY) | Units passing first inspection / Units started | Also called First Time Quality (FTQ) |
| DPMO | (Defects / (Units x Opportunities)) x 1,000,000 | Defects Per Million Opportunities; Six Sigma measure |
| Cost of Poor Quality (COPQ) | Internal failure + External failure + Appraisal + Prevention costs | Internal = scrap/rework; External = warranty, returns |
| Return Rate | Customer returns / Units shipped | Warranty return rate |
| Supplier Quality Rejection Rate | Rejected PO lines / Total PO lines | For incoming material |
| Escaped Defects | Customer complaints / Units shipped | Defects not caught before shipment |

### SPC (Statistical Process Control)

SPC requires time-series measurement data per control characteristic per process step:

```
FACT_SPC_MEASUREMENT
  measurement_id (PK)
  material_id (FK)
  work_center_id (FK)
  inspection_characteristic_id (FK)  -- e.g., "Shaft Diameter"
  production_order_id (FK)
  measurement_datetime
  measured_value
  lower_control_limit
  upper_control_limit
  lower_spec_limit
  upper_spec_limit
  is_out_of_control    -- UCL/LCL violation
  is_out_of_spec       -- USL/LSL violation
```

---

## 5. Supply Chain and Procurement

### Procurement Lifecycle

```
Purchase Requisition (PR)
    --> Request for Quotation (RFQ) [optional]
        --> Purchase Order (PO)
            --> Goods Receipt (GR)  [inventory increases]
                --> Invoice Receipt (IR)  [accounts payable]
                    --> Payment
```

### Supply Chain KPI Definitions

| KPI | Formula | Notes |
|-----|---------|-------|
| On-Time Delivery (OTD) | PO lines delivered on/before confirmed date / Total PO lines | Supplier OTD = supplier perspective; Customer OTD = outbound delivery |
| Fill Rate | Lines shipped complete / Lines ordered | Order line fill rate most common |
| Lead Time | GR date - PO creation date | Also: PO date - PR date (procurement lead time) |
| Lead Time Variability | Std deviation of lead time | High variability drives safety stock |
| Safety Stock | z-score x std_dev(demand) x sqrt(lead_time) | Statistical safety stock formula |
| Inventory Turns | COGS / Average Inventory Value | Higher = leaner; benchmark varies by industry |
| Days of Supply (DOS) | Average Inventory / Average Daily Demand | Complement of turns; in days |
| Spend by Category | SUM(PO line amount) grouped by commodity/category | Spend cube: category x supplier x plant x time |

### Supplier Scorecard Dimensions

| Dimension | Metrics | Weight (Typical) |
|-----------|---------|-----------------|
| Quality | Rejection rate, PPM defects, corrective action closure rate | 30% |
| Delivery | OTD %, lead time accuracy, emergency delivery rate | 30% |
| Responsiveness | Quote response time, issue resolution time | 20% |
| Cost | Price variance vs market, cost reduction YoY | 20% |

### Logistics Data Model

```
FACT_SHIPMENT
  shipment_id (PK)
  delivery_id (FK)           -- Outbound or inbound delivery
  carrier_id (FK)            -- DIM_CARRIER
  ship_from_plant_id (FK)
  ship_to_customer_id (FK)
  ship_date_key (FK)
  expected_delivery_date_key (FK)
  actual_delivery_date_key (FK)
  weight_kg
  volume_cbm
  freight_cost_usd
  freight_mode              -- (TL, LTL, Parcel, Ocean, Air, Rail)
  incoterm_code             -- (EXW, FCA, FOB, CIF, DAP, DDP)
  on_time_flag
  days_late                 -- negative = early
  carrier_performance_score
```

---

## 6. Warehouse and Inventory

### Inventory Stock Types

| Stock Type | Description | Can Be Used for Production? |
|------------|-------------|----------------------------|
| Unrestricted | Available, passes quality | Yes |
| Quality Inspection (QI) | Awaiting inspection decision | No (until released) |
| Blocked | Failed inspection or flagged | No |
| In Transit | Moving between plants | No |
| Consignment | Customer/supplier-owned on-site | Depends on terms |
| Reserved | Allocated to a production order | Technically yes (already reserved) |

**Critical:** Never aggregate across stock types without explicit business intent. A dashboard showing "total inventory" must define which stock types are included.

### Goods Movement Types (SAP Reference)

| Movement Type | Description |
|---------------|-------------|
| 101 | Goods Receipt for Purchase Order |
| 201 | Goods Issue to Cost Center |
| 261 | Goods Issue to Production Order |
| 311 | Transfer Posting between storage locations |
| 501 | Receipt without reference (adjustment) |
| 551 | Goods Issue for Scrapping |
| 601 | Delivery to Customer (Outbound) |
| 641 | Transfer Order / Stock Transfer |

### Warehouse Operations Flow

```
Inbound: Receive --> Unload --> Quality Check --> Putaway --> Unrestricted Stock
Internal: Pick --> Stage --> Issue to Production --> Confirmation
Outbound: Pick Order --> Pack --> Label --> Load --> Ship --> Delivery Confirmation
```

### Inventory KPI Definitions

| KPI | Formula | Notes |
|-----|---------|-------|
| Inventory Turns | Annual COGS / Average Inventory Value | Higher = leaner; 8-12x typical for manufacturing |
| Days of Supply | Avg Inventory Value / (Annual COGS / 365) | Lower = leaner |
| Dead Stock | Stock with zero movement > N days | Define N based on business (90, 180, 365 days) |
| Obsolescence Risk | Value of stock where demand < min reorder qty | Drives inventory write-off |
| Pick Accuracy | Orders picked correctly / Total orders picked | Warehouse quality metric |
| ABC Classification | A = top 80% of value, B = next 15%, C = last 5% | Value-based segmentation |
| XYZ Classification | X = stable demand, Y = variable, Z = sporadic | Demand variability segmentation |

---

## 7. IoT and Connected Products

### IoT Data Architecture Pattern

```
Equipment (Physical)
    --> Edge Device / PLC / Controller
        --> Edge Gateway (local aggregation, filtering)
            --> Cloud Ingestion (Kafka / Kinesis / Event Hub)
                --> Raw Time-Series Store (Bronze layer)
                    --> Cleansed / Resampled (Silver layer)
                        --> Aggregated KPIs (Gold layer)
                            --> Predictive Models / Dashboards
```

### Equipment Telemetry Data Model

```
DIM_EQUIPMENT (asset master)
  equipment_id (PK)
  serial_number
  model_number
  equipment_type          -- (forklift, conveyor, CNC machine, compressor)
  manufacturer
  manufacture_date
  install_date
  plant_id (FK)
  customer_id (FK)        -- If deployed at customer site (fleet)
  warranty_expiry_date
  expected_life_years

FACT_TELEMETRY (time-series, high volume)
  telemetry_id (PK or composite)
  equipment_id (FK)
  event_timestamp         -- Partition key
  metric_name             -- (engine_temp_c, battery_soc_pct, runtime_hours, fault_code)
  metric_value_numeric
  metric_value_text       -- For categorical signals
  data_quality_flag       -- (GOOD, SUSPECT, BAD)
  source_system
```

**Partitioning strategy:** Partition FACT_TELEMETRY by date (daily or monthly) and equipment_type. Without partitioning, queries against high-frequency sensor data become full scans.

### OEE for Connected Equipment

For IoT-enabled machines, OEE can be computed from telemetry rather than manual entries:

| OEE Component | IoT Signal |
|---------------|------------|
| Availability | Machine state signal (running / idle / fault / planned stop) |
| Performance | Actual cycle count vs theoretical cycle count per time unit |
| Quality | Good parts counter vs total parts counter (from part presence sensor) |

### Predictive Maintenance Signals

| Asset Type | Relevant Signals | Failure Mode |
|------------|-----------------|--------------|
| Electric Motor | Vibration (x/y/z), temperature, current draw | Bearing wear, winding failure |
| Hydraulic System | Pressure, flow rate, fluid temperature, fluid level | Seal leak, pump wear |
| Battery (Forklift) | State of charge, charge cycles, temperature | Capacity degradation |
| Conveyor | Belt tension, motor current, speed | Belt slip, motor overload |
| CNC Spindle | Vibration, temperature, tool load | Tool wear, spindle bearing |

### Fleet Analytics (Lift Truck / Material Handling)

Specific to Crown Equipment and similar manufacturers -- equipment deployed at customer sites:

```
FACT_FLEET_UTILIZATION (grain: equipment + day)
  utilization_key (PK)
  equipment_id (FK)
  customer_id (FK)          -- Site where deployed
  date_key (FK)
  hours_available
  hours_operating
  hours_idle
  hours_charging            -- Electric forklifts
  hours_fault
  utilization_pct           -- hours_operating / hours_available
  distance_traveled_km
  loads_handled_count
  fault_count
  fault_codes_array         -- List of fault codes encountered
```

---

## 8. Dealer / Distribution Networks

### Dealer Hierarchy

```
Region (e.g., North America)
  --> District (e.g., Midwest)
      --> Dealer (e.g., ABC Equipment Co.)
          --> Branch Location (if multi-location dealer)
```

### Dealer Scorecard Dimensions

| Dimension | Metrics |
|-----------|---------|
| Sales Performance | Units sold vs quota, revenue vs prior year, market share |
| Service Quality | Customer satisfaction (CSAT/NPS), first-time fix rate, mean time to repair |
| Parts Business | Parts fill rate, parts revenue, parts gross margin |
| Inventory Management | Aged inventory %, demo units, rental fleet utilization |
| Financial Health | Payment terms adherence, credit utilization |

### Dealer Analytics Data Model

```
FACT_DEALER_SALES (grain: dealer + model + month)
  dealer_sales_key (PK)
  dealer_id (FK)            -- DIM_DEALER
  product_model_id (FK)     -- DIM_PRODUCT
  date_key (FK)
  units_sold
  revenue_usd
  gross_margin_usd
  quota_units
  prior_year_units_sold
  order_type                -- (New, Used, Rental)

FACT_DEALER_FLEET (grain: equipment unit)
  fleet_key (PK)
  equipment_id (FK)         -- DIM_EQUIPMENT
  dealer_id (FK)
  customer_id (FK)          -- End customer
  sale_date_key (FK)
  warranty_expiry_date_key (FK)
  fleet_age_months          -- Calculated
  in_service_flag
```

---

## 9. ERP Data Structures

### SAP Module Reference

| SAP Module | Full Name | Key Data |
|------------|-----------|---------|
| PP | Production Planning | Production orders (AUFK, AFKO), BOM (STKO/STPO), Routing (PLPO), Confirmations (AFRU) |
| MM | Materials Management | Material master (MARA/MARC), BOM (STKO/STPO), PO (EKKO/EKPO), GR (MSEG/MKPF) |
| QM | Quality Management | Inspection lots (QALS), Results (QAMV), Notifications (QMEL), Usage decisions (QAVE) |
| PM | Plant Maintenance | Functional locations (IFLOT), Equipment (EQUI), Orders (AUFK PM type), Notifications (QMEL) |
| SD | Sales & Distribution | Sales orders (VBAK/VBAP), Deliveries (LIKP/LIPS), Customer master (KNA1) |
| WM/EWM | Warehouse Management | Transfer orders (LTAP), Storage types (T301), Quants (LQUA) |

### IFS Applications Reference

| IFS Module | Key Objects |
|------------|-------------|
| Manufacturing | Work Order, Operation, Work Center, Part Structure (BOM) |
| Inventory | Part, Inventory Part, Storage Location, Inventory Transaction |
| Purchasing | Purchase Order, Purchase Order Line, Supplier |
| Quality | Quality Notification, Inspection, Non-Conformance Report |
| Maintenance | Equipment Object, Maintenance Order, Failure Code |

### BAAN / Infor LN Reference

| BAAN / Infor LN Module | Key Objects |
|------------------------|-------------|
| Manufacturing | Production Order, Bill of Material, Routing, Work Center |
| Inventory | Item, Warehouse, Lot, Inventory Transaction |
| Procurement | Purchase Order, Receipt |
| Quality | Quality Notification, Inspection Order |

### Cross-ERP Integration Challenges

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| Terminology mismatch | "Work Order" in IFS = "Production Order" in SAP | Document canonical term in MDM layer |
| Granularity differences | SAP confirms by operation; IFS may confirm at order level | Define reporting grain at order level for comparability |
| Status code differences | Different status values for same lifecycle stage | Map to canonical status in integration layer |
| Identifier gaps | ERP item number != MDM material number | Maintain cross-reference table in integration layer |
| Calendar differences | Some ERPs use fiscal periods natively | Convert to calendar dates in Bronze/Silver layer |
| Multi-currency | SAP stores amounts in document currency + local currency; BAAN may differ | Standardize to reporting currency in Gold layer |

---

## 10. Manufacturing KPI Reference

### Production KPIs

| KPI | Formula | Benchmark (Discrete Mfg) |
|-----|---------|--------------------------|
| OEE | Availability x Performance x Quality | World-class: >85%; Typical: 60-75% |
| Availability | (Planned Time - Unplanned Downtime) / Planned Time | Target: >90% |
| Performance | Actual Output / Theoretical Output | Target: >95% |
| Quality Rate | Good Units / Total Units | Target: >99% |
| Schedule Attainment | Actual Production / Scheduled Production | Target: >95% |
| Scrap Rate | Scrap Qty / Total Started | Target: <1% (varies by process) |
| Throughput | Good Units / Hour (or Day) | Process-specific |

### Quality KPIs

| KPI | Formula | Notes |
|-----|---------|-------|
| DPMO | (Defects / Units x Opportunities) x 1,000,000 | Six Sigma: 3.4 DPMO at 6-sigma |
| Sigma Level | Derived from DPMO | 3-sigma (66,807 DPMO) to 6-sigma (3.4 DPMO) |
| First Pass Yield | Conforming units / Total units started | Without rework |
| Cost of Poor Quality (COPQ) | Internal failure + External failure costs | Typically 5-25% of revenue for manufacturers |
| Warranty Return Rate | Units returned under warranty / Units sold | Varies by product and industry |
| Supplier PPM | Supplier defects per million parts received | Target: <100 PPM |

### Supply Chain KPIs

| KPI | Formula | Benchmark |
|-----|---------|-----------|
| On-Time Delivery (OTD) | On-time lines / Total lines | Target: >95% |
| Fill Rate | Lines shipped complete / Lines ordered | Target: >97% |
| Inventory Turns | COGS / Average Inventory | 8-12x typical for manufacturing |
| Days of Supply | Avg Inventory / Daily COGS | 30-45 days typical |
| Perfect Order Rate | Orders delivered: on-time + complete + damage-free + correct invoice | Target: >95% |

### Maintenance KPIs

| KPI | Formula | Notes |
|-----|---------|-------|
| MTBF | Total Uptime / Number of Failures | Mean Time Between Failures; higher = better |
| MTTR | Total Repair Time / Number of Repairs | Mean Time to Repair; lower = better |
| PM Compliance | PMs completed on schedule / PMs planned | Target: >95% |
| Maintenance Cost per Unit | Total maintenance cost / Units produced | Tracks maintenance efficiency |

### Customer / Fleet KPIs

| KPI | Formula | Notes |
|-----|---------|-------|
| Net Promoter Score (NPS) | % Promoters - % Detractors | Survey-based; -100 to +100 |
| Fleet Uptime | Hours available - fault hours / Hours available | For connected equipment |
| Parts Fill Rate | Parts orders shipped complete / Parts orders received | Dealer parts service level |
| Mean Time to Service (MTTS) | Avg time from service request to resolution | Field service responsiveness |

---

## 11. Data Modeling Patterns

### Star Schema for Manufacturing

**Shared Dimensions across manufacturing facts:**

| Dimension | Reused By |
|-----------|-----------|
| DIM_MATERIAL | Production, Quality, Inventory, Supply Chain |
| DIM_PLANT | Production, Quality, Inventory, Supply Chain |
| DIM_DATE | All facts |
| DIM_WORK_CENTER | Production, Quality, IoT |
| DIM_CUSTOMER | Sales, Quality notifications, Fleet |
| DIM_VENDOR | Procurement, Quality (supplier) |
| DIM_EMPLOYEE | Production confirmations, Quality stewardship |

### Multi-Plant Modeling

Two options:

**Option A: Plant as dimension** (recommended for <20 plants)
- Single fact table with plant_id FK
- Filtering and rollup handled by BI tool
- Simple; works well with Medallion architecture

**Option B: Separate fact tables per plant region**
- Only needed for extreme data volumes or regulatory separation
- Requires UNION ALL views for cross-plant reporting
- Adds complexity; avoid unless volume demands it

### Unit of Measure Conversion

Always maintain a conversion table:

```
DIM_UOM
  uom_id (PK)
  uom_code        -- EA, KG, L, M, M2, M3, etc.
  uom_description
  uom_category    -- (Mass, Volume, Length, Area, Quantity)

FACT_UOM_CONVERSION
  from_uom_id (FK)
  to_uom_id (FK)
  conversion_factor   -- to_qty = from_qty * conversion_factor
  material_id (FK)    -- NULL for universal; non-null for material-specific (e.g., density)
```

**Do not hardcode UOM conversions in SQL.** Store in FACT_UOM_CONVERSION and join at query time or in the Silver layer transformation.

### Factory Calendar vs Fiscal Calendar

```
DIM_DATE (fiscal)
  date_key (PK)
  calendar_date
  fiscal_year
  fiscal_quarter
  fiscal_period
  fiscal_week

DIM_FACTORY_DATE (production)
  factory_date_key (PK)
  calendar_date
  plant_id (FK)
  shift_number        -- 1, 2, 3
  shift_start_time
  shift_end_time
  is_working_day
  is_holiday
  planned_capacity_hours
```

Joins between production facts (factory calendar) and financial facts (fiscal calendar) require explicit bridge or date-key alignment.

---

## 12. Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Shop floor data latency ignored | IoT/MES data arrives late; daily aggregations miss same-day production | Use late-arrival pattern with restatement window |
| Flat BOM assumption | Single-level join misses sub-assembly costs and MRP requirements | Always design for recursive / multi-level explosion |
| Aggregating across inventory stock types | Unrestricted + QI + Blocked != "available inventory" | Always filter to specific stock types; document which types included |
| OEE as a single formula | Computing OEE without Availability x Performance x Quality breakdown hides root causes | Compute all three components separately; expose in fact table |
| Dirty ERP data ignored | SAP/IFS data has null material numbers, missing plant codes, and test transactions | Implement data quality rules in Silver layer; flag and quarantine bad records |
| Dashboard before data model | Building reports before defining grain, dimensions, and KPIs creates technical debt | Define fact table grain and KPI formulas first; validate with business stakeholders |
| Timezone complexity underestimated | Global manufacturing data: plant in Shanghai + warehouse in Germany + HQ in Ohio | Store all timestamps in UTC; convert to local time in BI layer with plant timezone lookup |
| Ignoring revision history | Using current BOM/routing for historical production orders changes historical costs | Snapshot BOM/routing at production order creation; store historical snapshots |
| Single currency in multi-currency | Global manufacturers transact in EUR, USD, CNY; reporting in USD requires conversion | Store document currency + exchange rate; convert to reporting currency in Gold layer |
| MES data without master data link | Shop floor telemetry has machine IDs that don't match ERP equipment numbers | Build and maintain machine-to-ERP cross-reference in integration layer |

---

## Quick Reference

### OEE Calculation Worked Example

```
Shift: 8 hours = 480 minutes
Planned Stops: 30 min (breaks, planned maintenance)
Planned Production Time: 450 minutes

Unplanned Downtime: 45 minutes
Run Time: 405 minutes

Availability = 405 / 450 = 90.0%

Ideal Cycle Time: 1 minute per unit
Actual Output: 350 units
Performance = (350 x 1) / 405 = 86.4%

Total Units: 350
Defect Units: 7
Good Units: 343
Quality = 343 / 350 = 98.0%

OEE = 90.0% x 86.4% x 98.0% = 76.2%
```

### SAP Production Order Tables

| Table | Description |
|-------|-------------|
| AUFK | Order master (all order types) |
| AFKO | PP order header |
| AFPO | PP order item |
| AFVC | PP order operations |
| AFRU | PP confirmations |
| RESB | Reservations / BOM components on order |
| MSEG | Material document segments (goods movements) |
| MKPF | Material document header |

### Common Manufacturing SQL Patterns

```sql
-- OEE by work center and shift (from FACT_OEE_DAILY)
SELECT
    wc.work_center_code,
    wc.work_center_description,
    d.fiscal_year,
    d.fiscal_period,
    AVG(f.availability_pct)   AS avg_availability,
    AVG(f.performance_pct)    AS avg_performance,
    AVG(f.quality_pct)        AS avg_quality,
    AVG(f.oee_pct)            AS avg_oee,
    SUM(f.good_output_qty)    AS total_good_output,
    SUM(f.scrap_qty)          AS total_scrap
FROM fact_oee_daily f
JOIN dim_work_center wc ON f.work_center_id = wc.work_center_id
JOIN dim_date d         ON f.date_key = d.date_key
WHERE d.fiscal_year = 2025
GROUP BY wc.work_center_code, wc.work_center_description,
         d.fiscal_year, d.fiscal_period
ORDER BY avg_oee ASC;  -- Worst performers first

-- Supplier OTD last 90 days
SELECT
    v.vendor_name,
    COUNT(*) AS total_po_lines,
    SUM(CASE WHEN f.actual_gr_date <= f.promised_delivery_date THEN 1 ELSE 0 END) AS on_time_lines,
    ROUND(
        SUM(CASE WHEN f.actual_gr_date <= f.promised_delivery_date THEN 1 ELSE 0 END)
        * 100.0 / COUNT(*), 1
    ) AS otd_pct
FROM fact_purchase_order_line f
JOIN dim_vendor v   ON f.vendor_id = v.vendor_id
JOIN dim_date d     ON f.gr_date_key = d.date_key
WHERE d.calendar_date >= CURRENT_DATE - INTERVAL '90 days'
  AND f.actual_gr_date IS NOT NULL
GROUP BY v.vendor_name
ORDER BY otd_pct ASC;
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Manufacturing skill | /Users/donmccarty/ali-ai/skills/ali-manufacturing/SKILL.md |
| Manufacturing expert | /Users/donmccarty/ali-ai/agents/ali-manufacturing-expert.md |

---

## References

- APICS CPIM Body of Knowledge (production planning, MRP, inventory)
- ISO 9001 / IATF 16949 (quality management systems)
- OEE Industry Standard -- SEMI E10 (equipment performance)
- SAP Help Portal: PP, MM, QM module documentation
- IFS Applications documentation
- BAAN / Infor LN documentation
- ISA-95 (Enterprise-Control System Integration standard for MES)
- Crown Equipment Corporation RFP context: SAP, IFS, BAAN, Quipware, Apache Iceberg, Starburst, WhereScape, Power BI

---

**Document Version:** 1.0
**Last Updated:** 2026-02-18
**Maintained By:** ALI AI Team
