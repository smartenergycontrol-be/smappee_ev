# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Home Assistant custom integration for Smappee EV charging stations that provides advanced control and monitoring capabilities beyond the official Smappee integration. The integration supports multiple connectors on the same charging station and uses both REST API calls and MQTT streaming for real-time updates.

## Architecture

### Core Components

- **Integration Setup**: `__init__.py` - Manages setup and teardown of the integration, creates API clients for station and connectors
- **Config Flow**: `config_flow.py` - Multi-step configuration flow that discovers service locations and connectors
- **Data Coordinator**: `coordinator.py` - Single source of truth for data, handles REST API polling and MQTT data merging
- **MQTT Gateway**: `mqtt_gateway.py` - Real-time MQTT client for live charging station updates
- **API Client**: `api_client.py` - Command interface for interacting with Smappee API
- **OAuth**: `oauth.py` - Handles authentication and token refresh with Smappee API

### Data Flow

1. **Initial Setup**: Config flow authenticates, discovers service locations and charging devices
2. **REST Snapshot**: Coordinator fetches initial state via REST API calls  
3. **MQTT Streaming**: Live updates via MQTT using service location UUID as credentials
4. **Data Merging**: Coordinator merges MQTT updates with REST data and notifies entities

### Entity Structure

The integration creates entities per connector:
- **Sensors**: `charging_state`, `evcc_state`, `evcc_status` 
- **Controls**: `charging_mode` (select), `max_charging_speed` (number), charging buttons
- **Switches**: `connector_available`, `evcc_charging_control`
- **Station-level**: LED brightness control, MQTT diagnostics

## API Configuration

### Credentials
Store API credentials in `.secrets` file (already in gitignore):
```
api_client_id: your_client_id
api_secret: your_secret
username: your_username  
password: your_password
```

### API Endpoints
- **Base URL**: `https://app1pub.smappee.net/dev/v3`
- **OAuth**: `https://app1pub.smappee.net/dev/v1/oauth2/token`
- **MQTT**: `mqtt.smappee.net:443` (uses service location UUID as username/password)

## Key Issues Identified

### 1. Already Configured Error (config_flow.py:208-209)

**Problem**: When adding multiple connectors from the same charging station, the config flow uses the station UUID as unique_id, causing "already_configured" error on the second connector.

**Root Cause**: 
```python
await self.async_set_unique_id(station["uuid"])  # Same for all connectors
self._abort_if_unique_id_configured()
```

**Solution**: Use a combination of station UUID and connector-specific data for unique_id, or rework the flow to configure all connectors in a single config entry.

### 2. EVCC State Calculation Logic (coordinator.py:576-583)

**Problem**: EVCC state calculation only works correctly for one charging station connector.

**Analysis**: The issue is in `_derive_evcc_letter()` and `_update_evcc()` methods:
- EVCC state derived from `iec_status` field in MQTT payload
- Logic at coordinator.py:335-339 extracts first character (A/B/C/E/F)
- State is connector-specific but may have data collision issues

**MQTT Data Structure**: 
```json
{
  "iecStatus": {
    "previous": "B2", 
    "current": "B1"
  }
}
```

**Current Logic**:
```python
def _derive_evcc_letter(iec: str | None, _charging_state: str | None = None) -> str | None:
    if not iec:
        return None
    first = iec.strip()[:1].upper()  # Takes first char: "B1" -> "B"
    return first if first in ("A", "B", "C", "E", "F") else None
```

## Development Commands

Based on the manifest.json, this integration:
- Uses `aiomqtt==2.4.0` for MQTT communication
- Has no specific build/test/lint commands defined
- Supports config flow for easy setup in Home Assistant UI

## Testing

### API Testing
```bash
# Test OAuth authentication
curl -X POST https://app1pub.smappee.net/dev/v1/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password&client_id=CLIENT_ID&client_secret=SECRET&username=USER&password=PASS"

# Get service locations
curl -H "Authorization: Bearer ACCESS_TOKEN" https://app1pub.smappee.net/dev/v3/servicelocation

# Get smart devices for a location
curl -H "Authorization: Bearer ACCESS_TOKEN" https://app1pub.smappee.net/dev/v3/servicelocation/SERVICE_LOCATION_ID/smartdevices
```

### MQTT Testing
MQTT streams data to topic patterns like:
```
servicelocation/{UUID}/etc/carcharger/acchargingcontroller/v1/devices/{DEVICE_UUID}/property/chargingstate
servicelocation/{UUID}/power
```

## File Structure Patterns

- `custom_components/smappee_ev/` - Main integration code
- `docs/` - Documentation including EVCC, openEMS, emhass integration guides  
- `translations/` - UI translations (en, nl)
- `services.yaml` - Service definitions for Home Assistant

## Important Notes

- The integration disables REST polling after MQTT connection to avoid rate limiting
- Multiple connector support is the key differentiator from the official integration
- EVCC state calculation is critical for integration with external energy management systems
- Uses service location UUID as both MQTT username and password for authentication