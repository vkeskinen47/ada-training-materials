# ADE for Data Engineers — AI-Assisted Development

Exercises for a self-paced training in which data engineers develop an Agile Data Engine (ADE) data warehouse together with **ADA**, the Agile Data Agent.

**Read the exercises:** https://vkeskinen47.github.io/ada-training-materials/

## What the training is about

In the basic training *ADE for Data Engineers*, participants build a data pipeline by hand: NYC taxi data from staging through a Data Vault to a publish layer. The company in the business case, PackageDelivery, was considering taxi operations, but the decision was left open.

This training finishes the story. PackageDelivery's own fleet system, FLEETOPS, shows when its vans stand idle. The fleet data uses the same NYC taxi zones as the taxi data, so the two businesses meet in the existing data warehouse. The training ends with a decision memo: in which boroughs should PackageDelivery offer taxi service with its idle vans — and where does the data not support a recommendation?

ADA does the construction: it writes the YAML, runs the `ada` commands and talks to ADE. The participant's job is to:

- **Specify** — business questions, rules and decisions, precise enough for an agent to act on
- **Review** — designs, plans, SQL and numbers, and reject them when they are wrong
- **Verify** — results against checkpoints, known figures and their own hypotheses

## Exercises

| # | Exercise | Duration |
|---|---|---|
| 1 | [ADA Setup and New Source to Staging](Exercise_1_ADA_Setup_and_Staging.md) | 2 h |
| 2 | [Diagnose and Fix a Failed Load](Exercise_2_Diagnose_and_Fix.md) | 45 min |
| 3 | [Design the Model with ADA](Exercise_3_Design_the_Model.md) | 1 h |
| 4 | [Generate and Load the Raw Data Vault](Exercise_4_Raw_Data_Vault.md) | 1 h 30 min |
| 5 | [Publish Layer and Business Rules](Exercise_5_Publish_and_Business_Rules.md) | 1 h 30 min |
| 6 | [Analysis and Decision](Exercise_6_Analysis_and_Decision.md) | 1 h |

## Prerequisites

- *ADE for Data Engineers* (basic training) completed
- Python 3.10 or newer, Git and VS Code (or another supported AI tool) installed
- An AI coding assistant subscription, for example GitHub Copilot or Claude Code
- API keys and network access to the training environment, provided by the trainer

## About this repository

The exercises are published as web pages with GitHub Pages and embedded in the learning platform. Trainer material and answer keys are not stored here.

