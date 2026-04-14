# CRM Agent — Leads & Segmentation

## Overview

Handles everything related to contacts, persona-based segmentation, and campaign delivery logging in HubSpot. Falls back to simulation mode if no HubSpot token is configured.

## Agents

### ContactSyncAgent
Upserts contacts into HubSpot with persona metadata (`persona_type` custom property).

### SegmentationAgent
Adds contacts to persona-specific static lists in HubSpot.

### DispatchAgent
Logs campaign send events per persona (real HubSpot email or simulated send record).

## Status

> 🚧 To be designed — see [README.md](./README.md) for pipeline context.
