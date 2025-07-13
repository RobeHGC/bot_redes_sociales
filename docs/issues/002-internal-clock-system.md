# Internal Clock System (Laptop Deployment)

## Overview
Implement a timezone-aware internal clock system that manages daily content generation with intelligent recovery from downtime periods.

## Description
The internal clock system is the heart of Nadia Bot's autonomous operation. It manages scheduling for content generation, handles timezone operations for Mexico City (America/Mexico_City), provides intelligent recovery from missed days, and maintains persistent state across bot restarts.

## Acceptance Criteria

### Core Functionality
- [ ] **Timezone Management**: Native America/Mexico_City timezone support with automatic DST handling
- [ ] **Daily Scheduling**: Default 10:00 AM CDMX trigger for content generation
- [ ] **State Persistence**: Save/load bot state from `data/bot_state.json` across restarts
- [ ] **Recovery Intelligence**: Handle missed days with user-friendly prompts

### Recovery Logic Implementation
- [ ] **1 Day Missed**: Prompt "¿Genero el de ayer ahora?" with immediate generation option
- [ ] **2-7 Days Missed**: Offer "¿Resumen de trending topics?" for catch-up content
- [ ] **7+ Days Missed**: Auto-continue with normal schedule, no special handling

### State Management
- [ ] **Persistent Storage**: JSON-based state storage that survives system restarts
- [ ] **State Recovery**: Automatic state validation and repair on startup
- [ ] **Backup Strategy**: Daily state backups with corruption recovery
- [ ] **Configuration**: User-configurable schedule times and timezone preferences

## Technical Implementation

### Required Components
```python
# Core scheduler with APScheduler
class InternalClock:
    def __init__(self, timezone='America/Mexico_City'):
        self.scheduler = AsyncIOScheduler(timezone=timezone)
        self.state_file = 'data/bot_state.json'
        self.default_time = '10:00'  # 10:00 AM CDMX
    
    async def schedule_daily_generation(self):
        # Daily content generation job
    
    async def handle_missed_days(self):
        # Intelligent recovery logic
    
    async def save_state(self):
        # Persist current state
    
    async def load_state(self):
        # Restore state from file
```

### State Schema
```json
{
    "last_generation": "2024-01-15T10:00:00-06:00",
    "consecutive_days": 5,
    "total_posts": 45,
    "user_timezone": "America/Mexico_City",
    "schedule_time": "10:00",
    "recovery_preferences": {
        "auto_catchup": true,
        "max_catchup_days": 7
    }
}
```

### Error Handling
- [ ] **Clock Drift**: Monitor and correct system clock inconsistencies
- [ ] **Scheduler Failures**: Automatic scheduler restart with state preservation
- [ ] **File Corruption**: State file validation and backup recovery
- [ ] **Timezone Changes**: Handle DST transitions and timezone updates

## Dependencies
- **Before**: None (foundational component)
- **After**: #003 (Hybrid Command System), #004 (Article Processor)

## Testing Requirements
- [ ] **Unit Tests**: All time calculations and state management functions
- [ ] **Integration Tests**: End-to-end scheduling and recovery scenarios
- [ ] **Edge Cases**: DST transitions, leap years, system clock changes
- [ ] **Performance**: State save/load operations under 100ms
- [ ] **Reliability**: 7-day continuous operation without failures

## Configuration Options
```yaml
clock_settings:
  timezone: "America/Mexico_City"
  daily_time: "10:00"
  recovery_enabled: true
  max_catchup_days: 7
  state_backup_interval: 24h
  corruption_recovery: true
```

## User Interface
```bash
# Configuration commands
/config timezone America/Mexico_City
/config schedule 10:00
/config recovery auto

# Status commands  
/status clock
/status schedule

# Manual triggers
/generar ahora
/recuperar ayer
```

## Monitoring & Metrics
- [ ] **Uptime Tracking**: Monitor scheduler availability and execution
- [ ] **Accuracy Metrics**: Track timing precision and schedule adherence
- [ ] **Recovery Success**: Measure recovery operation success rates
- [ ] **Performance**: Log state operation times and memory usage

## Labels
- `core-system`
- `high-priority` 
- `infrastructure`
- `timezone`
- `persistence`

## Estimated Effort
**Story Points**: 5
**Timeline**: 1-2 weeks

## Definition of Done
- [ ] Clock system runs continuously with 99% uptime
- [ ] All recovery scenarios tested and working
- [ ] State persistence survives system restarts
- [ ] Timezone handling accurate including DST
- [ ] Performance targets met (<100ms state operations)
- [ ] Comprehensive test coverage (>90%)
- [ ] Documentation complete with configuration examples
- [ ] Monitoring dashboard operational