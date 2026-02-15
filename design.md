# Smart Irrigation Scheduler - Design Document

## Executive Summary

The Smart Irrigation Scheduler is an AI-powered autonomous system that optimizes agricultural water usage through intelligent decision-making. By combining IoT sensors, weather forecasting, and evapotranspiration calculations, the system reduces water waste by 20-40% compared to traditional fixed-schedule irrigation while maintaining optimal crop health.

## System Overview

### Vision
Create an autonomous irrigation system that acts as an intelligent agent, perceiving environmental conditions, reasoning about water needs, and executing precise irrigation decisions to promote sustainable farming practices.

### Core Value Proposition
- **Water Conservation**: Reduce water usage by 20-40% through intelligent scheduling
- **Autonomous Operation**: Minimal human intervention required after initial setup
- **Crop-Specific Intelligence**: Adapts to different crops and growth stages
- **Weather-Aware**: Prevents unnecessary irrigation before rain events
- **Real-Time Monitoring**: Live dashboard with instant control and feedback

---

## Design Principles

### 1. Agentic AI Architecture
The system embodies true agentic properties:
- **Perception**: Multi-source sensing (soil sensors + weather APIs)
- **Memory**: Hybrid storage (edge flash + cloud time-series)
- **Reasoning**: ET0-based calculation with multi-factor decision logic
- **Planning**: Precise volume and duration scheduling
- **Action**: Physical actuation through relay control
- **Feedback**: Closed-loop verification and impact analysis

### 2. Hybrid Intelligence
- **Edge Intelligence**: Local safety logic and auto-calibration on ESP32
- **Cloud Intelligence**: Centralized AI agent with weather integration
- **Human-in-the-Loop**: Manual override capability with priority handling

### 3. Fail-Safe Design
- Local relay control continues even if cloud connection drops
- Auto-calibration adapts to different soil types
- Safety thresholds prevent over-watering
- Manual override always takes precedence

### 4. Scalability
- Modular architecture supports multiple zones
- Cloud-based storage enables historical analysis
- API-driven design allows integration with farm management systems

---

## System Architecture

### Layer Model (9-Layer Agentic Architecture)

#### 1. Environment Layer
**Components**: Soil, crops, weather conditions
**Role**: The physical world the system interacts with

#### 2. Perception Layer
**Components**: YL-69 soil moisture sensor, OpenWeatherMap API
**Role**: Gather real-time environmental data
**Specifications**:
- Sensor Range: 0-4095 (12-bit ADC)
- Calibrated Output: 0-100% moisture
- Polling Interval: 5 seconds
- Weather Update: Every 60 seconds

#### 3. Edge Control Layer
**Components**: ESP32 microcontroller
**Role**: Local processing and safety logic
**Key Features**:
- Auto-calibration algorithm for soil type adaptation
- Smoothing filter (5-sample moving average)
- Local relay control with safety limits
- LCD display for status feedback
**Specifications**:
- Processor: ESP32 (240MHz dual-core)
- Memory: 520KB SRAM
- Connectivity: WiFi 802.11 b/g/n

#### 4. Communication Layer
**Protocol**: HTTP/MQTT to ThingsBoard
**Data Flow**:
- Telemetry Upload: Every 10 seconds
- Command Polling: Every 2 seconds
- Attribute Sync: On-demand
**Latency**: <2 seconds for command execution

#### 5. Cloud Storage Layer
**Platform**: ThingsBoard (Community/Cloud Edition)
**Data Types**:
- Time-series telemetry (soil moisture, pump state)
- Device attributes (crop type, field size, thresholds)
- Commands (pump control, manual override)
**Retention**: Configurable (default 30 days)

#### 6. Memory & Context Layer
**Components**: 
- Short-term: Current sensor state and pump status
- Long-term: Historical trends in ThingsBoard
- Semantic: Crop knowledge base (Kc coefficients, growth stages)
**Data Structures**:
```typescript
SystemState {
  soilMoisturePercent: number
  pumpState: "ON" | "OFF"
  manualOverride: boolean
  lastUpdated: Date | null
}
```

#### 7. Reasoning Layer (AI Agent)
**Location**: Python decision engine (`decision_core.py`)
**Algorithm**: ET0-based water demand calculation
**Decision Formula**:
```
Net Demand = (ET0 × Kc × (1 - SoilFactor × 0.8)) - ExpectedRain
```
**Decision Matrix**:
| Soil Moisture | Rain Probability | Action |
|---------------|------------------|--------|
| < 30% | Any | Irrigate |
| 30-70% | < 30% | Calculate |
| 30-70% | > 60% | Skip |
| > 70% | Any | Skip |

#### 8. User Interface Layer
**Technologies**: 
- Next.js 15 (React 18)
- Tailwind CSS + Framer Motion
- Recharts for data visualization
**Features**:
- Real-time dashboard with live updates
- Manual pump control with override
- Configurable settings (crop, soil, field size)
- Weekly impact report with savings visualization
- Rain alerts and thinking trace display

#### 9. Action Layer
**Components**: Relay module, water pump
**Specifications**:
- Relay: 5V trigger, 250V/10A switching
- Pump: Submersible, 12V DC
- Flow Rate: Configurable based on field size
**Safety**:
- Maximum run time: 600 seconds per cycle
- Minimum interval: 300 seconds between cycles

---

## Data Flow Architecture

### Primary Flow (Autonomous Mode)
```
1. Sensor → ESP32 (calibration + smoothing)
2. ESP32 → ThingsBoard (telemetry upload)
3. ThingsBoard ← AI Agent (fetch data)
4. Weather API → AI Agent (forecast data)
5. AI Agent → ThingsBoard (command push)
6. ThingsBoard → ESP32 (command poll)
7. ESP32 → Relay → Pump (actuation)
8. Pump → Soil (irrigation)
9. Soil → Sensor (feedback loop)
```

### Secondary Flow (User Interaction)
```
1. User → Dashboard (configuration/override)
2. Dashboard → API Route (Next.js)
3. API Route → ThingsBoard (attribute update)
4. ThingsBoard → ESP32 (sync)
5. ESP32 → Relay (immediate action)
```

---

## Component Design

### ESP32 Firmware (`esp32_irrigation.ino`)

**Responsibilities**:
- Sensor reading and calibration
- Local display management
- Cloud communication
- Relay control with safety checks

**Key Functions**:
```cpp
calibrateSensor(int raw, String soilType) -> float
  // Maps 0-4095 to 0-100% based on soil characteristics
  
smoothMoisture(float current) -> float
  // 5-sample moving average filter
  
controlPump(String command, int duration)
  // Safety-checked relay actuation
  
syncWithCloud()
  // Bidirectional data exchange with ThingsBoard
```

**State Machine**:
- INIT: WiFi connection + sensor calibration
- IDLE: Monitoring + telemetry upload
- WATERING: Pump active + safety monitoring
- MANUAL: User override mode

### AI Decision Engine (`decision_core.py`)

**Class**: `SmartIrrigationAgent`

**Core Methods**:
```python
analyze_and_decide(current_moisture: float) -> Dict
  # Multi-factor decision logic
  # Returns: action, duration, reasoning_trace
  
get_weather_forecast() -> Dict
  # OpenWeatherMap API integration
  # Returns: temperature, rain_probability, conditions
  
calibrate_moisture(raw_value: int, soil_type: str) -> float
  # Soil-specific calibration curves
  
push_decision_to_thingsboard(decision_data)
  # Command execution via RPC
```

**Decision Algorithm**:
1. Fetch current soil moisture from ThingsBoard
2. Retrieve weather forecast (rain probability, temperature)
3. Load crop parameters (type, growth stage, Kc coefficient)
4. Calculate ET0 (reference evapotranspiration)
5. Compute net water demand
6. Apply decision rules
7. Generate reasoning trace
8. Push command to ThingsBoard

**Crop Knowledge Base**:
```python
CROP_COEFFICIENTS = {
    "Rice (Paddy)": {
        "Vegetative": 1.1,
        "Reproductive": 1.25,
        "Ripening": 1.0
    },
    "Wheat": {
        "Vegetative": 0.7,
        "Reproductive": 1.15,
        "Ripening": 0.4
    },
    "Sugarcane": {
        "Vegetative": 0.8,
        "Reproductive": 1.25,
        "Ripening": 0.7
    },
    "Cotton": {
        "Vegetative": 0.35,
        "Reproductive": 1.2,
        "Ripening": 0.6
    }
}
```

### Next.js Dashboard

**Architecture**: App Router with API Routes

**Key Routes**:
- `/` - Main dashboard page
- `/api/command` - Pump control endpoint
- `/api/ui/state` - System state endpoint
- `/api/upload` - Telemetry ingestion

**Components**:
```
Dashboard.tsx
├── DailySchedule.tsx (current plan + reasoning)
├── ImpactReport.tsx (weekly savings chart)
├── RainAlert.tsx (weather warnings)
├── SettingsForm.tsx (crop/soil configuration)
└── ThinkingTrace.tsx (AI decision steps)
```

**State Management** (`lib/store.ts`):
```typescript
// In-memory state with getters/setters
- soilMoisturePercent
- pumpState
- manualOverride
- lastUpdated
```

**Services**:
- `irrigation-engine.ts`: Core AI logic (TypeScript port)
- `weather-service.ts`: Open-Meteo API integration
- `thingsboard.ts`: ThingsBoard API client

---

## API Design

### Internal APIs (Next.js)

#### POST /api/command
**Purpose**: Control pump state
**Request**:
```json
{
  "command": "ON" | "OFF",
  "manual": boolean
}
```
**Response**:
```json
{
  "success": true,
  "state": "ON" | "OFF"
}
```

#### GET /api/ui/state
**Purpose**: Fetch current system state
**Response**:
```json
{
  "soilMoisturePercent": 45,
  "pumpState": "OFF",
  "manualOverride": false,
  "lastUpdated": "2026-02-15T10:30:00Z"
}
```

### External APIs

#### OpenWeatherMap API
**Endpoint**: `https://api.openweathermap.org/data/2.5/forecast`
**Usage**: 5-day weather forecast
**Rate Limit**: 60 calls/minute (free tier)

#### Open-Meteo API (Alternative)
**Endpoint**: `https://api.open-meteo.com/v1/forecast`
**Usage**: Real-time weather data
**Rate Limit**: Unlimited (free)

#### ThingsBoard REST API
**Base URL**: `https://demo.thingsboard.io/api`
**Endpoints**:
- `POST /auth/login` - Authentication
- `POST /v1/{token}/telemetry` - Upload data
- `GET /v1/{token}/attributes` - Fetch config
- `POST /v1/{token}/rpc` - Send commands

---

## Data Models

### Telemetry Data
```json
{
  "ts": 1708000000000,
  "values": {
    "soil_moisture": 45.2,
    "pump_state": "OFF",
    "calibrated_raw": 2048,
    "temperature": 28.5
  }
}
```

### Device Attributes
```json
{
  "crop_type": "Rice (Paddy)",
  "growth_stage": "Vegetative",
  "soil_type": "loam",
  "field_size_hectares": 1.5,
  "moisture_threshold_low": 30,
  "moisture_threshold_high": 70,
  "manual_override": false
}
```

### Decision Output
```json
{
  "date": "2026-02-15",
  "time": "10:30:00",
  "action": "Irrigate",
  "amount_liters_per_hectare": 45000,
  "total_amount_liters": 67500,
  "reasoning_trace": [
    {
      "step": 1,
      "description": "Assess Crop Needs",
      "result": "SAFE",
      "details": "Rice (Paddy) (Vegetative, Kc: 1.1). Demand: 7.15 mm."
    }
  ],
  "savings_vs_fixed": 37500,
  "weather_summary": "28°C, 15% Rain",
  "soil_status": "45% Moisture",
  "pump_state_recommendation": "ON"
}
```

---

## Algorithm Details

### ET0 Calculation (Simplified)
```
ET0 = Base Reference Evapotranspiration (6.5 mm/day)
Kc = Crop Coefficient (varies by crop and stage)
Water Demand (mm) = ET0 × Kc
```

### Soil Factor Calculation
```
if moisture < 30%:
    soilFactor = 0 (needs full water)
elif moisture > 70%:
    soilFactor = 1 (needs no water)
else:
    soilFactor = (moisture - 30) / 40 (linear interpolation)
```

### Net Demand Formula
```
requiredMM = waterDemandMM × (1 - soilFactor × 0.8) - expectedRainMM
requiredMM = max(0, requiredMM)
```

### Volume Conversion
```
Liters per Hectare = requiredMM × 10,000
Total Liters = Liters per Hectare × Field Size (hectares)
Pump Duration (seconds) = Total Liters / Flow Rate (L/s)
```

### Auto-Calibration Algorithm
```cpp
// Soil-specific calibration curves
float calibrateSensor(int raw, String soilType) {
  float dryValue, wetValue;
  
  if (soilType == "clay") {
    dryValue = 3500; wetValue = 1500;
  } else if (soilType == "sand") {
    dryValue = 3800; wetValue = 1200;
  } else { // loam (default)
    dryValue = 3600; wetValue = 1400;
  }
  
  float percent = 100 - ((raw - wetValue) / (dryValue - wetValue) * 100);
  return constrain(percent, 0, 100);
}
```

---

## Security Design

### Authentication
- ThingsBoard device tokens (unique per device)
- API key for weather services
- No hardcoded credentials in firmware (use environment variables)

### Authorization
- Device-level access control in ThingsBoard
- Manual override requires user authentication (future enhancement)
- Rate limiting on API endpoints

### Data Privacy
- No personally identifiable information collected
- Telemetry data encrypted in transit (HTTPS/MQTTS)
- Local storage option for sensitive deployments

### Safety Mechanisms
- Maximum pump runtime limits (600s)
- Minimum interval between cycles (300s)
- Emergency stop via manual override
- Watchdog timer on ESP32

---

## Performance Specifications

### Latency
- Sensor to Cloud: <5 seconds
- Command to Actuation: <2 seconds
- Dashboard Update: Real-time (WebSocket future enhancement)

### Throughput
- Telemetry Upload: 6 messages/minute
- Command Processing: 30 commands/minute
- API Response Time: <200ms (p95)

### Reliability
- Uptime Target: 99.5%
- Sensor Accuracy: ±5% after calibration
- Weather Forecast Accuracy: Dependent on provider (typically 80-90%)

### Scalability
- Single Device: 1-10 hectares
- Multi-Zone: Up to 50 devices per ThingsBoard instance
- Cloud Storage: Unlimited with proper retention policies

---

## Deployment Architecture

### Development Environment
```
Local Machine
├── ESP32 (USB connected)
├── Python Agent (localhost)
├── Next.js Dev Server (localhost:3000)
└── ThingsBoard Demo (cloud)
```

### Production Environment
```
Field Deployment
├── ESP32 + Sensors (WiFi connected)
├── Cloud Server (Python agent)
│   └── Cron job (every 60s)
├── Web Server (Next.js production)
│   └── Vercel/AWS/DigitalOcean
└── ThingsBoard Cloud/Self-hosted
```

### Hardware Setup
```
ESP32 Connections:
├── GPIO 34 → YL-69 Analog Out
├── GPIO 21 → LCD SDA
├── GPIO 22 → LCD SCL
├── GPIO 26 → Relay IN
└── 5V/GND → Power distribution
```

---

## Testing Strategy

### Unit Testing
- Sensor calibration functions
- Decision logic with mock data
- API endpoint handlers
- State management functions

### Integration Testing
- ESP32 ↔ ThingsBoard communication
- AI Agent ↔ Weather API
- Dashboard ↔ Backend APIs
- End-to-end command flow

### Field Testing
- Sensor accuracy validation
- Pump actuation reliability
- WiFi connectivity stability
- Battery backup (if applicable)

### Simulation Testing
- Mock weather scenarios
- Extreme soil conditions
- Network failure recovery
- Concurrent user interactions

---

## Monitoring & Observability

### Metrics
- Soil moisture trends
- Pump runtime statistics
- Water savings calculations
- API response times
- Error rates

### Logging
- ESP32: Serial output for debugging
- Python Agent: Structured logging (INFO/ERROR)
- Next.js: Server-side request logs
- ThingsBoard: Audit logs

### Alerts
- Critical soil dryness (<20%)
- Pump failure detection
- Network connectivity loss
- Weather API failures

---

## Future Enhancements

### Phase 2 Features
- Multi-zone support (multiple fields)
- Mobile app (React Native)
- SMS/Email notifications
- Historical analytics dashboard
- Machine learning for ET0 prediction

### Phase 3 Features
- Soil nutrient sensors (NPK)
- Drone imagery integration
- Predictive maintenance
- Community data sharing
- Carbon footprint tracking

### Advanced AI
- Reinforcement learning for optimal scheduling
- Computer vision for crop health assessment
- Anomaly detection for sensor failures
- Yield prediction models

---

## Sustainability Impact

### Water Conservation
- Target: 20-40% reduction vs fixed schedules
- Mechanism: Precision irrigation based on actual need
- Measurement: Weekly impact reports

### Energy Efficiency
- Reduced pump runtime = lower electricity consumption
- Solar power compatibility (future)
- Optimized cloud processing

### Environmental Benefits
- Reduced groundwater depletion
- Prevention of soil erosion from over-watering
- Lower fertilizer runoff
- Improved crop health and yield

### Economic Impact
- Lower water bills for farmers
- Reduced labor costs (automation)
- Improved crop quality
- ROI: Typically 6-12 months

---

## Compliance & Standards

### Agricultural Standards
- FAO guidelines for irrigation efficiency
- Local water usage regulations
- Crop-specific best practices

### Technical Standards
- IEEE 802.11 (WiFi)
- MQTT 3.1.1 / HTTP 1.1
- JSON data format (RFC 8259)
- ISO 8601 timestamps

### Safety Standards
- Electrical safety (relay/pump)
- Water quality standards
- Environmental protection regulations

---

## Glossary

- **ET0**: Reference Evapotranspiration - the amount of water that would evaporate from a reference crop
- **Kc**: Crop Coefficient - multiplier that adjusts ET0 for specific crops
- **YL-69**: Resistive soil moisture sensor model
- **ThingsBoard**: Open-source IoT platform for device management
- **Relay**: Electrically operated switch for controlling high-power devices
- **Telemetry**: Automated measurement and transmission of data
- **RPC**: Remote Procedure Call - method for executing commands on remote devices
- **Agentic AI**: AI system with perception, reasoning, and autonomous action capabilities

---

## References

- FAO Irrigation and Drainage Paper 56 (ET0 calculation)
- OpenWeatherMap API Documentation
- ThingsBoard Documentation
- ESP32 Technical Reference Manual
- Next.js 15 Documentation

---

**Document Version**: 1.0  
**Last Updated**: February 15, 2026  
**Author**: AI Ignite Hackathon Team  
**Status**: Production Ready
