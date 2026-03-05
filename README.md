# ali-plugins

Org-wide Claude Code plugin packs for Aliunde projects.

## Register the marketplace

    claude plugin marketplace add dmccarty-ali/ali-plugins

## Install a pack

    claude plugin install ali-core@ali-plugins
    claude plugin install ali-data-warehouse@ali-plugins

## Available packs

- ali-core — Core agents and skills for all Aliunde projects (7 agents, 10 skills)
- ali-data-warehouse — Snowflake, dbt, and data warehouse agents and skills (7 agents, 7 skills)

## Relationship to ali-ai

This repo is the distribution copy. The source of truth for all agents and skills is
the ali-ai repo (github.com/dmccarty-ali/ali-ai). When ali-ai releases a new version,
packs in this repo are updated to match.
