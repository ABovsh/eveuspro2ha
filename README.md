# Eveus Pro integration with Home Assistant

## Table of Contents
1. [Disclaimer](#disclaimer)
2. [Features & Improvements](#features--improvements)
3. [Quick Start Guide](#quick-start-guide)
4. [Configuration Steps](#configuration-steps)
   - [Add credentials](#1-add-credentials)
   - [Add sensors](#2-add-sensors)
   - [Add current regulator](#3-add-current-regulator)
   - [Add input numbers](#4-add-input-numbers)
   - [Add switches](#5-add-switches)
   - [Validate and restart](#6-validate-and-restart)
5. [UI Setup](#ui-setup)
   - [Basic Entity Card](#basic-entity-card)
   - [Advanced UI Elements](#advanced-ui-elements)
     - [Slider Controls](#slider-controls)
     - [Action Buttons](#action-buttons)
6. [Notifications](#notifications)
   - [Session Start Notification](#session-start-notification)
   - [Current Change Notification](#current-change-notification)
   - [Session Complete Notification](#session-complete-notification)
7. [SOC Calculation Guide](#soc-calculation-guide)

## Disclaimer
The code within this repository comes with no guarantee, the use of this code is your responsibility.

I take NO responsibility and/or liability for how you choose to use information available here. By using any of the files available in this repository, you understand that you are AGREEING TO USE AT YOUR OWN RISK.

## Features & Improvements
1. **Enhanced Sensor Functionality**
   - Added descriptions and proper device classes
   - Expanded status interpretation with error handling
   - Added sub-status mapping for both error and operational states

2. **Improved Charging Controls**
   - Enhanced current control with validation
   - Fixed "Stop charging" logic with error states
   - Added proper confirmations and status feedback

3. **Advanced SOC Tracking**
   - Added kWh and percentage SOC monitoring
   - Integrated loss factor correction
   - Added capacity limits and validation

4. **Smart Charging Features**
   - Added "Time to Target" calculation
   - Multiple status messages
   - Insufficient power detection

5. **Energy Monitoring**
   - Counter A and B integration
   - Cost tracking in local currency
   - Comprehensive error handling

6. **Flexible Configuration**
   - Secrets for credentials
   - Configurable battery capacity
   - Adjustable loss factor
   - Note: EVEUS IP ADDRESS requires manual YAML editing

7. **Counter Management**
   - Reset functionality with confirmation
   - Timeout handling
   - Status monitoring

8. **UI Enhancements**
   - Theme-aware styling
   - Mobile-friendly controls
   - Haptic feedback support
   - Status-based color coding

9. **Robust Error Handling**
   - Comprehensive error detection
   - Connection timeouts
   - Value validation
   - Detailed status messages

**Known Limitations:**
- Eveus IP Address configuration requires direct YAML editing
## Quick Start Guide
1. **Prepare Your Environment**
   - Ensure your Home Assistant is up and running
   - Make sure you have access to configuration files
   - Know your Eveus charger's IP address

2. **Basic Setup (5 minutes)**
   - Add credentials to secrets.yaml
   - Copy sensor configurations to configuration.yaml
   - Add controllers and inputs
   - Restart Home Assistant

3. **UI Configuration (5 minutes)**
   - Add basic entity card
   - Configure sliders and buttons
   - Set up notifications

4. **Final Configuration (5 minutes)**
   - Set your EV battery capacity
   - Configure correction factor
   - Test the setup

## Configuration Steps

### 1. Add credentials
Add the following to your `/config/secrets.yaml` file:
```yaml
eveus_username: "your charger username"
eveus_password: "your charger password"
```
### 2. Add sensors
Add these sensors to your /config/configuration.yaml file:
```
# Additional sensors for EVSE
- platform: rest
  name: EVSE Eveus
  resource: http://<EVEUS_IP_ADDRESS>/main
  username: !secret eveus_username
  password: !secret eveus_password
  method: POST
  json_attributes:
    - evseEnabled
    - state
    - subState
    - currentSet
    - curDesign
    - typeEvse
    - typeRelay
    - ground
    - timerType
    - timeZone
    - systemTime
    - timeMsg
    - timeLimitS
    - energyLimitS
    - moneyLimitS
    - suspendErrors
    - suspendLimits
    - oneCharge
    - totalEnergy
    - sessionTime
    - sessionEnergy
    - sessionMoney
    - sessionStarted
    - IEM1
    - IEM1_money
    - IEM2
    - IEM2_money
    - curMeas1
    - curMeas2
    - curMeas3
    - voltMeas1
    - voltMeas2
    - voltMeas3
    - powerMeas
    - temperature1
    - temperature2
    - vBat
    - STA_IP_Addres
  value_template: "EVSE_Eveus"
  scan_interval: 60 # Update every N seconds

- platform: template
  sensors:
    # Session time sensor
    evse_eveus_sessiontime:
      friendly_name: "Session Time"
      unique_id: evse_eveus_sessiontime
      value_template: >
        {{ state_attr('sensor.evse_eveus', 'sessionTime')|int(0) }}

    # Session duration sensor with formatting
    evse_eveus_newsessiontime:
      friendly_name: "Session Duration"
      unique_id: evse_eveus_newsessiontime
      value_template: >-
        {% set uptime = state_attr('sensor.evse_eveus', 'sessionTime')|int(0) %}
        {% if uptime > 0 %}
          {% set days = uptime // 86400 %}
          {% set hours = (uptime % 86400) // 3600 %}
          {% set minutes = (uptime % 3600) // 60 %}
          {{ '%dd ' % days if days > 0 }}{{ '%dh ' % hours if hours > 0 }}{{ '%dm' % minutes if minutes > 0 }}
        {% else %}
          No active session
        {% endif %}

    # Sensor for EVSE enabled status
    evse_eveus_enabled:
      value_template: "{{ state_attr('sensor.evse_eveus', 'evseEnabled') }}"
      friendly_name: "EVSE Enabled"

    # Sensor for EVSE state with human-readable description
    evse_eveus_state:
      value_template: >
        {% set mapper =  {
          0: 'Startup',
          1: 'System Test',
          2: 'Standby',
          3: 'Connected',
          4: 'Charging',
          5: 'Charge Complete',
          6: 'Paused',
          7: 'Error'
        } %}
        {% set state_num = state_attr('sensor.evse_eveus', 'state') %}
        {{ mapper[state_num] if state_num in mapper else 'Unknown' }}
      friendly_name: "Charger State"

    # Sensor for EVSE sub-state with human-readable description
    evse_eveus_substate:
      value_template: >
        {% set state_num = state_attr('sensor.evse_eveus', 'state') | int(0) %}
        {% set substate_num = state_attr('sensor.evse_eveus', 'subState') %}

        {% if substate_num is none %}
          unknown
        {% else %}
          {% set substate_num = substate_num | int(0) %}
          {% if state_num == 7 %}
            {% set mapper = {
              0: 'No Error',
              1: 'Grounding Error',
              2: 'Current Leak High',
              3: 'Relay Error',
              4: 'Current Leak Low',
              5: 'Box Overheat',
              6: 'Plug Overheat',
              7: 'Pilot Error',
              8: 'Low Voltage',
              9: 'Diode Error',
              10: 'Overcurrent',
              11: 'Interface Timeout',
              12: 'Software Failure',
              13: 'GFCI Test Failure',
              14: 'High Voltage'
            } %}
          {% else %}
            {% set mapper = {
              0: 'No Limits',
              1: 'Limited by User',
              2: 'Energy Limit',
              3: 'Time Limit',
              4: 'Cost Limit',
              5: 'Schedule 1 Limit',
              6: 'Schedule 1 Energy Limit',
              7: 'Schedule 2 Limit',
              8: 'Schedule 2 Energy Limit',
              9: 'Waiting for Activation',
              10: 'Paused by Adaptive Mode'
            } %}
          {% endif %}
          {{ mapper.get(substate_num, 'Unknown') }}
        {% endif %}
      friendly_name: "Charger Substate"

    # Sensor for current set in amperes
    evse_eveus_currentset:
      value_template: "{{ state_attr('sensor.evse_eveus', 'currentSet') }}"
      friendly_name: "Current set"
      unit_of_measurement: "A"

    # Sensor for current designed in amperes
    evse_eveus_curdesign:
      value_template: "{{ state_attr('sensor.evse_eveus', 'curDesign') }}"
      unit_of_measurement: "A"
      friendly_name: "Current designed"

    # Sensor for voltage measurement 1
    evse_eveus_voltmeas1:
      value_template: "{{ state_attr('sensor.evse_eveus', 'voltMeas1') | round(1) }}"
      unit_of_measurement: "V"
      friendly_name: "Voltage"

    # Sensor for current measurement 1
    evse_eveus_curmeas1:
      value_template: "{{ state_attr('sensor.evse_eveus', 'curMeas1') | round(1) }}"
      unit_of_measurement: "A"
      friendly_name: "Charger Current"

    # Sensor for power measurement in watts
    evse_eveus_powermeas:
      value_template: "{{ state_attr('sensor.evse_eveus', 'powerMeas') | round(1) }}"
      unit_of_measurement: "W"
      friendly_name: "Charger Power"

    # Sensor for temperature of the box
    evse_eveus_temperature1:
      value_template: "{{ state_attr('sensor.evse_eveus', 'temperature1') }}"
      unit_of_measurement: "°C"
      friendly_name: "Temp of box"

    # Sensor for temperature of the plug
    evse_eveus_temperature2:
      value_template: "{{ state_attr('sensor.evse_eveus', 'temperature2') }}"
      unit_of_measurement: "°C"
      friendly_name: "Temp of plug"

    # Sensor for ground status
    evse_eveus_ground:
      value_template: "{% if state_attr('sensor.evse_eveus', 'ground') == 1 %}Yes{% else %}No{% endif %}"
      friendly_name: "Ground"

    # Sensor for ground control status
    evse_eveus_groundctrl:
      value_template: "{% if state_attr('sensor.evse_eveus', 'groundCtrl') == 2 %}Yes{% else %}No{% endif %}"
      friendly_name: "Ground control"

    # Sensor for adaptive mode voltage
    evse_eveus_aivoltage:
      value_template: "{{ state_attr('sensor.evse_eveus', 'aiVoltage') }}"
      unit_of_measurement: "V"
      friendly_name: "Adaptive mode voltage"

    # Sensor for adaptive mode current
    evse_eveus_aicurrent:
      value_template: "{{ state_attr('sensor.evse_eveus', 'aiModecurrent') }}"
      unit_of_measurement: "A"
      friendly_name: "Adaptive mode current"

    # Sensor for session energy
    evse_eveus_sessionenergy:
      value_template: "{{ (state_attr('sensor.evse_eveus', 'sessionEnergy') | float) | round(1) }}"
      unit_of_measurement: "kWh"
      friendly_name: "Session Energy"

    # Sensor for session time
    evse_eveus_sessiontime:
      value_template: "{{ state_attr('sensor.evse_eveus', 'sessionTime') }}"
      unit_of_measurement: "s"
      friendly_name: "Session Time"

    # Sensor for total energy
    evse_eveus_totalenergy:
      value_template: "{{ (state_attr('sensor.evse_eveus', 'totalEnergy') | float) | round(1) }}"
      unit_of_measurement: "kWh"
      friendly_name: "Total Energy"

    # Sensor for AI status
    evse_eveus_aistatus:
      value_template: "{{ state_attr('sensor.evse_eveus', 'aiStatus') }}"
      friendly_name: "Adaptive mode status"

    # Sensor for battery voltage
    evse_eveus_vbat:
      value_template: "{{ state_attr('sensor.evse_eveus', 'vBat') }}"
      unit_of_measurement: "V"
      friendly_name: "Battery voltage"

    # Sensor for system time
    evse_eveus_systemtime:
      value_template: "{{ state_attr('sensor.evse_eveus', 'systemTime') }}"
      friendly_name: "System Time"

    # Sensor for current time
    current_time:
      friendly_name: "Current Time"
      value_template: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"

    # Sensor for Counter A Energy
    evse_eveus_counter_a_energy:
      friendly_name: "Counter A Energy (kWh)"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM1') | round(2) }}"
      unit_of_measurement: "kWh"

    # Sensor for Counter A Cost
    evse_eveus_counter_a_cost:
      friendly_name: "Counter A Cost"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM1_money') | round(2) }}"
      unit_of_measurement: "₴"

    # Sensor for Counter B Energy
    evse_eveus_counter_b_energy:
      friendly_name: "Counter B Energy (kWh)"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM2') | round(2) }}"
      unit_of_measurement: "kWh"

    # Sensor for Counter B Cost
    evse_eveus_counter_b_cost:
      friendly_name: "Counter B Cost"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM2_money') | round(2) }}"
      unit_of_measurement: "₴"

    # Sensor for EV State of Charge in kWh
    ev_soc_kwh:
      friendly_name: "EV State of Charge"
      unique_id: ev_soc_kwh
      device_class: energy
      unit_of_measurement: "kWh"
      value_template: >
        {% set entities = {
          'initial_soc': states('input_number.initial_ev_soc')|float(0),
          'max_capacity': states('input_number.ev_battery_capacity')|float(0),
          'energy_charged': states('sensor.evse_eveus_counter_a_energy')|float(0),
          'correction': states('input_number.ev_soc_correction')|float(0)
        } %}
        
        {% set is_valid = 
          entities.initial_soc >= 0 and
          entities.initial_soc <= 100 and
          entities.max_capacity > 0 and
          is_number(entities.energy_charged)
        %}
        
        {% if not is_valid %}
          unknown
        {% else %}
          {% set initial_kwh = (entities.initial_soc / 100) * entities.max_capacity %}
          {% set efficiency = (1 - entities.correction / 100) %}
          {% set charged_kwh = entities.energy_charged * efficiency %}
          {% set total_kwh = initial_kwh + charged_kwh %}
          {{ [0, [total_kwh, entities.max_capacity]|min]|max|round(2) }}
        {% endif %}

    # Sensor for EV State of Charge as percentage 
    ev_soc_percent:
      friendly_name: "EV State of Charge"
      unique_id: ev_soc_percent
      device_class: battery
      unit_of_measurement: "%"
      value_template: >
        {% set data = {
          'current_kwh': states('sensor.ev_soc_kwh')|float(-1),
          'max_capacity': states('input_number.ev_battery_capacity')|float(0)
        } %}
        
        {% if data.current_kwh >= 0 and data.max_capacity > 0 %}
          {% set percentage = (data.current_kwh / data.max_capacity * 100)|round(0) %}
          {{ [0, [percentage, 100]|min]|max }}
        {% else %}
          unknown
        {% endif %}

    # Sensor for Time to Target SOC
    evse_time_to_target_soc:
      friendly_name: "Time to Target SOC"
      unique_id: evse_time_to_target_soc
      value_template: >
        {% set vars = namespace() %}
        {% set vars.status = 'unknown' %}
    
        {% if not is_state('sensor.evse_eveus_state', 'Charging') %}
          {% set vars.status = 'Not charging' %}
        {% elif states('sensor.ev_soc_percent') in ['unknown', 'unavailable', 'none'] %}
          {% set vars.status = 'unknown' %}
        {% else %}
          {% set vars.current_soc = states('sensor.ev_soc_percent')|float(-1) %}
          {% if vars.current_soc < 0 or vars.current_soc > 100 %}
            {% set vars.status = 'unknown' %}
          {% else %}
            {% set vars.target_soc = states('input_number.target_soc')|float(0) %}
            {% if vars.target_soc <= vars.current_soc %}
              {% set vars.status = 'Target reached' %}
            {% else %}
              {% set vars.power_meas = states('sensor.evse_eveus_powermeas')|float(0) %}
              {% if vars.power_meas < 100 %}
                {% set vars.status = 'Insufficient power' %}
              {% else %}
                {% set vars.battery_capacity = states('input_number.ev_battery_capacity')|float(0) %}
                {% set vars.correction = states('input_number.ev_soc_correction')|float(0) %}
                
                {% set vars.remaining_kwh = (vars.target_soc - vars.current_soc) * vars.battery_capacity / 100 %}
                {% set vars.power_kw = vars.power_meas * (1 - vars.correction / 100) / 1000 %}
                {% set vars.total_minutes = (vars.remaining_kwh / vars.power_kw * 60)|round(0) %}
                
                {% if vars.total_minutes < 1 %}
                  {% set vars.status = 'Less than 1m' %}
                {% elif vars.total_minutes < 60 %}
                  {% set vars.status = vars.total_minutes ~ 'm' %}
                {% else %}
                  {% set vars.hours = (vars.total_minutes // 60)|int %}
                  {% set vars.minutes = (vars.total_minutes % 60)|int %}
                  {% set vars.status = vars.hours ~ 'h' ~ (vars.minutes > 0 and ' ' ~ vars.minutes ~ 'm' or '') %}
                {% endif %}
              {% endif %}
            {% endif %}
          {% endif %}
        {% endif %}
        {{ vars.status }}

```
### 3. Add current regulator
Add to your /config/configuration.yaml:
```
shell_command:
  evse_current_incr: "curl -s -u !secret eveus_username:!secret eveus_password -X POST -H 'Content-type: application/x-www-form-urlencoded' 'http://<EVEUS_IP_ADDRESS>/pageEvent' -d \"currentSet=$(($(curl -s -u !secret eveus_username:!secret eveus_password -X POST 'http://<EVEUS_IP_ADDRESS>/main' | jq '.currentSet')+1))\""
  evse_current_decr: "curl -s -u !secret eveus_username:!secret eveus_password -X POST -H 'Content-type: application/x-www-form-urlencoded' 'http://<EVEUS_IP_ADDRESS>/pageEvent' -d \"currentSet=$(($(curl -s -u !secret eveus_username:!secret eveus_password -X POST 'http://<EVEUS_IP_ADDRESS>/main' | jq '.currentSet')-1))\""
  evse_current_set: "curl -s -u !secret eveus_username:!secret eveus_password -X POST -H 'Content-type: application/x-www-form-urlencoded' 'http://<EVEUS_IP_ADDRESS>/pageEvent' -d \"currentSet={{ '%02d'|format(states('input_number.evse_eveus_current')|int) }}\""

```
### 4. Add input numbers
Add to your /config/configuration.yaml:
```
input_number:
  ev_battery_capacity:
    name: "EV Battery Capacity (kWh)"
    min: 10
    max: 100
    step: 1
    mode: slider
    unit_of_measurement: "kWh"
    icon: mdi:car

  initial_ev_soc:
    name: Initial EV State of Charge (%)
    min: 0
    max: 100
    step: 1
    mode: slider
    icon: mdi:BatteryCharging40

  ev_soc_correction:
    name: SOC Correction Factor (%)
    min: 0
    max: 10
    step: 0.1
    initial: 7.5
    mode: slider
    icon: mdi:TuneVariant

  target_soc:
    name: Target SOC (%)
    min: 80
    max: 100
    step: 10
    initial: 80
    mode: slider
    icon: mdi:battery-charging-high

  evse_eveus_current:
    name: Charger Set Current
    min: 8
    max: 16
    step: 1
    initial: 16
    mode: slider
    icon: mdi:current-dc
```
### 5. Add switches
Add to your /config/configuration.yaml:
```
command_line:
  - switch:
      name: "Eveus Reset Counter A"
      unique_id: evse_eveus_reset_counter_a
      icon: mdi:counter
      command_on: >
        curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        -H "Content-type: application/x-www-form-urlencoded" 
        "http://<EVEUS_IP_ADDRESS>/pageEvent" 
        -d "pageevent=rstEM1&rstEM1=0" 
        || echo "ERROR"
      command_off: >
        curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        -H "Content-type: application/x-www-form-urlencoded" 
        "http://<EVEUS_IP_ADDRESS>/pageEvent" 
        -d "pageevent=rstEM1&rstEM1=0" 
        || echo "ERROR"
      command_state: >-
        (curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        "http://<EVEUS_IP_ADDRESS>/main" 
        | jq -r "if .IEM1 != null then .IEM1 else \"ERROR\" end") 2>/dev/null || echo "ERROR"
      value_template: >-
        {% if value in ['ERROR', 'null', '', 'undefined'] %}
          false
        {% else %}
          {{ value | int != 0 }}
        {% endif %}

  - switch:
      name: "Charging Control"
      unique_id: evse_eveus_stop_charging
      icon: mdi:ev-station
      command_on: >
        curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        -H "Content-type: application/x-www-form-urlencoded" 
        "http://<EVEUS_IP_ADDRESS>/pageEvent" 
        -d "pageevent=evseEnabled&evseEnabled=1" 
        || echo "ERROR"
      command_off: >
        curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        -H "Content-type: application/x-www-form-urlencoded" 
        "http://<EVEUS_IP_ADDRESS>/pageEvent" 
        -d "pageevent=evseEnabled&evseEnabled=0" 
        || echo "ERROR"
      command_state: >-
        (curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        "http://<EVEUS_IP_ADDRESS>/main" 
        | jq -r "if .evseEnabled != null then .evseEnabled else \"ERROR\" end") 2>/dev/null || echo "ERROR"
      value_template: >-
        {% if value in ['ERROR', 'null', '', 'undefined'] %}
          false
        {% else %}
          {{ value == '1' }}
        {% endif %}

  - switch:
      name: "One Charge"
      unique_id: evse_eveus_one_charge
      icon: mdi:lightning-bolt
      command_on: >
        curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        -H "Content-type: application/x-www-form-urlencoded" 
        "http://<EVEUS_IP_ADDRESS>/pageEvent" 
        -d "pageevent=oneCharge&oneCharge=1" 
        || echo "ERROR"
      command_off: >
        curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        -H "Content-type: application/x-www-form-urlencoded" 
        "http://<EVEUS_IP_ADDRESS>/pageEvent" 
        -d "pageevent=oneCharge&oneCharge=0" 
        || echo "ERROR"
      command_state: >-
        (curl -s --connect-timeout 2 --max-time 5 -u !secret eveus_username:!secret eveus_password -X POST 
        "http://<EVEUS_IP_ADDRESS>/main" 
        | jq -r "if .oneCharge != null then .oneCharge else \"ERROR\" end") 2>/dev/null || echo "ERROR"
      value_template: >-
        {% if value in ['ERROR', 'null', '', 'undefined'] %}
          false
        {% else %}
          {{ value == '1' }}
        {% endif %}
```
### 6. Validate and restart
Go to Configuration -> Server Controls
Click CHECK CONFIGURATION button
If successful, click RESTART in Server Management section

## UI Setup
### Basic Entity Card
Create an entity card in your Home Assistant dashboard with this configuration:
```
type: entities
entities:
  - entity: sensor.ev_soc_percent
    name: SOC (%)
    icon: mdi:battery-charging-80
  - entity: sensor.ev_soc_kwh
    name: SOC (kWh)
    icon: mdi:battery
  - entity: sensor.evse_time_to_target_soc
    name: Time to Target
    icon: mdi:clock-time-four
  - type: divider
  - entity: sensor.evse_eveus_state
    name: Charger State
    icon: mdi:car-electric
  - entity: sensor.evse_eveus_substate
    name: Substate
    icon: mdi:car-cog
  - type: divider
  - entity: sensor.evse_eveus_powermeas
    name: Power (W)
    icon: mdi:flash
  - entity: sensor.evse_eveus_counter_a_energy
    name: Session Energy (kWh)
    icon: mdi:counter
  - type: divider
  - entity: input_number.evse_eveus_current
    name: Current (A)
    icon: mdi:current-ac
  - entity: input_number.initial_ev_soc
    name: Initial SOC (%)
    icon: mdi:calendar-clock
  - entity: input_number.target_soc
    name: Target SOC (%)
    icon: mdi:target
  - entity: input_number.ev_soc_correction
    name: SOC Correction (%)
    icon: mdi:chart-bell-curve
  - entity: input_number.ev_battery_capacity
    name: Battery Capacity (kWh)
    icon: mdi:car-battery
  - type: divider
  - entity: switch.evse_one_charge
    name: One Charge Mode
    icon: mdi:power-plug
  - entity: switch.evse_stop_charging
    name: Stop Charging
    icon: mdi:stop-circle
  - entity: switch.evse_reset_counter_a
    name: Reset Energy Counter
    icon: mdi:reload
show_header_toggle: false
```
### Advanced UI Elements
![Screenshot_20241231_213946_Home Assistant](https://github.com/user-attachments/assets/62b55d07-497f-44a0-bba6-66a3a80dad86)

- **Real-time EV data**: Displays current SOC (State of Charge), time to target SOC, and charging current.
- **Tap Actions**: 
  - **Single Tap**: Show more detailed information for each sensor (e.g., SOC, current).
  - **Long Tap**: **Set Initial SOC for long tap on the SOC button and Charge Current for the Current button**
- **Color-coded indicators**: Dynamically change colors based on sensor values for easy status recognition.
- **Interactive controls**: Toggle switches to reset the counter, start a one-time charge, or stop charging.

#### Installation

1. **Install Mushroom Cards**:
   Follow the [Mushroom Cards installation guide](https://github.com/piitaya/lovelace-mushroom).

2. **Copy the YAML Configuration**:
   Add the provided YAML code to your Home Assistant Lovelace UI.
```
type: grid
columns: 3
square: false
cards:
  - type: custom:mushroom-template-badge
    entity: sensor.ev_soc_percent
    icon: mdi:battery-charging
    color: |
      {% if is_state('sensor.evse_eveus_state', 'Charging') %}
        {% set soc = states('sensor.ev_soc_percent')|float(0) %}
        {% if soc < 20 %}red
        {% elif soc <= 50 %}orange
        {% else %}green{% endif %}
      {% else %}gray{% endif %}
    label: "{{ states('sensor.ev_soc_percent') }}%"
    content: SOC
    tap_action:
      action: more-info
    hold_action:
      action: more-info
      entity: input_number.initial_ev_soc
  - type: custom:mushroom-template-badge
    entity: sensor.evse_time_to_target_soc
    icon: mdi:timer-outline
    color: |
      {% if is_state('sensor.evse_time_to_target_soc', 'unavailable') %}
        gray
      {% else %}
        {% set time = states('sensor.evse_time_to_target_soc') %}
        {% set hours = time|regex_findall_index('\\d+h')|int(0) %}
        {% if hours < 2 %}green
        {% elif hours <= 6 %}orange
        {% else %}red{% endif %}
      {% endif %}
    label: "{{ states('sensor.evse_time_to_target_soc') }}"
    content: Time
    tap_action:
      action: more-info
  - type: custom:mushroom-template-badge
    entity: input_number.evse_eveus_current
    icon: mdi:flash
    color: |
      {% if is_state('sensor.evse_eveus_state', 'Charging') %}
        {% set current = states('input_number.evse_eveus_current')|float(0) %}
        {% if current > 12 %}
          red
        {% elif current > 8 %}
          orange
        {% else %}
          green
        {% endif %}
      {% else %}
        gray
      {% endif %}
    label: "{{ states('input_number.evse_eveus_current') }}A"
    content: Current
    tap_action:
      action: more-info
    hold_action:
      action: call-service
      service: input_number.set_value
      target:
        entity_id: input_number.evse_eveus_current
  - type: custom:mushroom-template-badge
    entity: switch.evse_reset_counter_a
    icon: mdi:backup-restore
    color: |
      {% if is_state('switch.evse_reset_counter_a', 'on') %}
        #3949AB
      {% else %}
        gray
      {% endif %}
    content: CounterA Rst
    tap_action:
      action: toggle
      confirmation:
        text: Reset counter A?
  - type: custom:mushroom-template-badge
    entity: switch.evse_one_charge
    icon: mdi:ev-station
    color: |
      {% if is_state('switch.evse_one_charge', 'on') %}
        #2E7D32
      {% else %}
        gray
      {% endif %}
    content: OneCharge
    tap_action:
      action: toggle
  - type: custom:mushroom-template-badge
    entity: switch.evse_stop_charging
    icon: mdi:stop-circle
    color: |
      {% if is_state('switch.evse_stop_charging', 'on') %}
        #C62828
      {% else %}
        gray
      {% endif %}
    content: StopCharge
    tap_action:
      action: toggle
      confirmation:
        text: Stop charging?
```
## Notifications
This section provides YAML configurations for setting up notifications in Home Assistant for various events during an EV charging session. Below are the available automations and their purposes. Use **Settings > Automations** to add these. 
![Screenshot_20241231_214300_Telegram](https://github.com/user-attachments/assets/d2bf4866-bf4e-4bc0-8415-e3c6d6d7b0e9)
### Session Start Notification
To notify when an EV charging session starts, use the YAML configuration provided in [301_EV_Charging_Started](https://github.com/ABovsh/eveuspro2ha/blob/main/301_EV_Charging_Started). This automation monitors when an EV charging session begins. Replace `<NOTIFICATION_SERVICE_NAME>` with your notification service name.
### Session Complete Notification
To notify when an EV charging session is completed, use the YAML configuration in [303_EV_Charging_Completed](https://github.com/ABovsh/eveuspro2ha/blob/main/303_EV_Charging_Completed). This automation detects when charging has finished. Replace `<NOTIFICATION_SERVICE_NAME>` with your notification service name.
### Current Change Notification
To notify when the EV charging current changes, use the YAML configuration in [302_EV_Charging_CurrentChanged](https://github.com/ABovsh/eveuspro2ha/blob/main/302_EV_Charging_CurrentChanged). This automation tracks fluctuations in charging current during the session. Replace `<NOTIFICATION_SERVICE_NAME>` with your notification service name.
## SOC Calculation Guide
### Initial Setup
1. **Set Battery Capacity** (one-time configuration)
    * Navigate to your dashboard
    * Find "Battery Capacity (kWh)" slider
    * Set your EV's actual battery capacity

2. **Configure Correction Factor**
    * Default value: 7.5%
    * Adjust based on your charging efficiency
    * Higher values account for more charging losses

### Before Each Charging Session
1. **Reset Counter A**
    * Press the "Reset" button
    * Confirm the reset action
    * Verify counter shows 0 kWh

2. **Set Initial SOC**
    * Use the "Init SOC" slider
    * Set to your car's current battery percentage
    * Ensure accurate starting point

3. **Optional: Adjust Target SOC**
    * Default is 80% (recommended for battery longevity)
    * Can be adjusted between 80-100%
    * Higher values may affect battery health
