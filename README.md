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
    # Sensor for session duration formatted as time
    evse_eveus_newsessiontime:
      friendly_name: "eveus session duration"
      value_template: >-
        {% set uptime = states.sensor.evse_eveus_sessiontime.state | int %}
        {% set years = uptime // 31536000 %}
        {% set months = (uptime % 31536000) // 2592000 %}
        {% set days = (uptime % 2592000) // 86400 %}
        {% set hours = (uptime % 86400) // 3600 %}
        {% set minutes = (uptime % 3600) // 60 %}
        {{ '%dy ' % years if years else '' }}{{ '%dm ' % months if months else '' }}{{ '%dd ' % days if days else '' }}{{ '%dh ' % hours if hours else '' }}{{ '%dm' % minutes if minutes else '' }}

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
        {% set initial_soc = states('input_number.initial_ev_soc') | float(-1) %}
        {% set max_capacity = states('input_number.ev_battery_capacity') | float(0) %}
        {% set energy_charged = states('sensor.evse_eveus_counter_a_energy') | float(0) %}
        {% set correction_factor = states('input_number.ev_soc_correction') | float(0) / 100 %}
        
        {# Enhanced validation with more specific error handling #}
        {% if not is_number(initial_soc) or not is_number(max_capacity) or not is_number(energy_charged) %}
          unknown
        {% elif initial_soc < 0 or initial_soc > 100 %}
          unknown
        {% elif max_capacity <= 0 %}
          unknown
        {% else %}
          {% set initial_kwh = (initial_soc / 100) * max_capacity %}
          {% set charged_kwh = energy_charged * (1 - correction_factor) %}
          {% set total_kwh = initial_kwh + charged_kwh %}
          {# Prevent showing less than 0 or more than max capacity #}
          {{ [0, [total_kwh, max_capacity] | min] | max | round(2) }}
        {% endif %}

    # Sensor for EV State of Charge as percentage
    ev_soc_percent:
      friendly_name: "EV State of Charge"
      unique_id: ev_soc_percent
      device_class: battery
      unit_of_measurement: "%"
      value_template: >
        {% set current_kwh = states('sensor.ev_soc_kwh') | float(-1) %}
        {% set max_capacity = states('input_number.ev_battery_capacity') | float(0) %}
        
        {# Enhanced validation #}
        {% if not is_number(current_kwh) or not is_number(max_capacity) %}
          unknown
        {% elif current_kwh < 0 or max_capacity <= 0 %}
          unknown
        {% else %}
          {# Ensure percentage stays between 0 and 100 #}
          {{ [0, [((current_kwh / max_capacity) * 100) | round(0), 100] | min] | max }}
        {% endif %}

    # Sensor for Time to Target SOC
    evse_time_to_target_soc:
      friendly_name: "Time to Target SOC"
      unique_id: evse_time_to_target_soc
      value_template: >
        {% set current_soc = states('sensor.ev_soc_percent') | float(-1) %}
        {% set target_soc = states('input_number.target_soc') | float(0) %}
        {% set charging_power = states('sensor.evse_eveus_powermeas') | float(0) %}
        {% set correction_factor = states('input_number.ev_soc_correction') | float(0) / 100 %}
        {% set max_capacity = states('input_number.ev_battery_capacity') | float(0) %}
        {% set is_charging = is_state('sensor.evse_eveus_state', 'Charging') %}
        
        {# Enhanced input validation #}
        {% if not is_number(current_soc) or not is_number(target_soc) or not is_number(max_capacity) %}
          unknown
        {% elif current_soc < 0 or current_soc > 100 %}
          unknown
        {% elif target_soc <= 0 or target_soc > 100 %}
          unknown
        {% elif max_capacity <= 0 %}
          unknown
        {% elif current_soc >= target_soc %}
          Target reached
        {% elif not is_charging %}
          Not charging
        {% elif charging_power < 100 %}
          Insufficient power
        {% else %}
          {% set remaining_kwh = (target_soc - current_soc) / 100 * max_capacity %}
          {% set effective_power_kw = charging_power * (1 - correction_factor) / 1000 %}
          {% set total_minutes = (remaining_kwh / effective_power_kw * 60) | round(0) %}
          
          {# Format output #}
          {% set days = (total_minutes // 1440) | int %}
          {% set hours = ((total_minutes % 1440) // 60) | int %}
          {% set minutes = (total_minutes % 60) | int %}
          
          {% if total_minutes < 1 %}
            Less than 1m
          {% elif total_minutes < 60 %}
            {{ minutes }}m
          {% else %}
            {{ days ~ 'd ' if days > 0 }}{{ hours ~ 'h ' if hours > 0 }}{{ minutes ~ 'm' if minutes > 0 }}
          {% endif %}
        {% endif %}

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
![Screenshot 2024-12-09 204029](https://github.com/user-attachments/assets/87db7d47-08ab-4098-b877-e57b3bdc6c25)
### Advanced UI Elements
#### Slider Controls
1. Install slider button card from this repo - https://github.com/mattieha/slider-button-card
2. Create sliders by using this code:
```
type: horizontal-stack
cards:
  - type: custom:slider-button-card
    entity: input_number.evse_eveus_current
    name: Current
    compact: true
    slider:
      direction: left-right
      background: gradient
      use_state_color: true
      show_track: true
      min: 8
      max: 16
      step: 1
    icon:
      show: true
      icon: mdi:flash
      tap_action:
        action: more-info
    show_name: true
    show_state: true
    show_attribute: false
    action_button:
      show: false
    style: |
      .card-header {
        font-size: 12px;  # Reduced font size for compact mode
        font-weight: bold;
      }
      .slider-button-card .slider {
        margin-top: 5px;  # Add space between the name and the slider
      }
  - type: custom:slider-button-card
    entity: input_number.initial_ev_soc
    name: Init SOC
    compact: true
    slider:
      direction: left-right
      background: gradient
      use_state_color: true
      show_track: true
      min: 0
      max: 100
      step: 1
    icon:
      show: true
      icon: mdi:battery
      tap_action:
        action: more-info
    show_name: true
    show_state: true
    show_attribute: false
    action_button:
      show: false
    style: |
      .card-header {
        font-size: 12px;  # Reduced font size for compact mode
        font-weight: bold;
      }
      .slider-button-card .slider {
        margin-top: 5px;  # Add space between the name and the slider
      }

```
![Screenshot 2024-12-09 205257](https://github.com/user-attachments/assets/2cab6384-b60c-4bec-bf51-7e5641d251e4)
#### Action Buttons
1. Install button-card from this repo - https://github.com/custom-cards/button-card
2. Create buttons by using this code:
```
type: horizontal-stack
cards:
  - type: custom:button-card
    entity: switch.evse_reset_counter_a
    name: Reset
    icon: mdi:reload
    size: 30%
    color_type: card
    color: "#3949AB"
    styles:
      card:
        - border-radius: 12px
        - height: 55px
        - padding: 4px
        - margin: 2px
      name:
        - font-size: 12px
        - font-weight: bold
        - padding-top: 4px
        - color: "#43749f"
      icon:
        - width: 24px
        - color: "#FFFFFF"
    tap_action:
      action: toggle
      confirmation:
        text: Reset the energy counter?
      haptic: light
    hold_action:
      action: more-info
      haptic: light
    state:
      - value: "off"
        styles:
          card:
            - background-color: transparent
          icon:
            - color: "#43749f"
          name:
            - color: "#43749f"
      - value: "on"
        styles:
          card:
            - opacity: 0.7
            - background-color: "#FF5722"
          name:
            - color: "#FFFFFF"
  - type: custom:button-card
    entity: switch.evse_one_charge
    name: Charge
    icon: mdi:ev-station
    size: 35%
    color_type: card
    color: "#2E7D32"
    tap_action:
      action: toggle
      haptic: light
    hold_action:
      action: more-info
      haptic: light
    styles:
      card:
        - border-radius: 12px
        - height: 55px
        - padding: 4px
        - margin: 2px
      name:
        - font-size: 12px
        - font-weight: bold
        - padding-top: 4px
        - color: "#FFFFFF"
      icon:
        - width: 30px
        - color: "#43749f"
    state:
      - value: "off"
        styles:
          card:
            - background-color: transparent
          icon:
            - color: "#43749f"
          name:
            - color: "#43749f"
      - value: "on"
        color: "#388E3C"
        styles:
          card:
            - background-color: "#2E7D32"
            - animation: pulse 2s infinite
          name:
            - color: "#FFFFFF"
          icon:
            - color: "#FFFFFF"
  - type: custom:button-card
    entity: switch.evse_stop_charging
    name: Stop
    icon: mdi:stop-circle-outline
    size: 35%
    color_type: card
    color: "#C62828"
    tap_action:
      action: toggle
      confirmation:
        text: Stop charging session?
      haptic: light
    hold_action:
      action: more-info
      haptic: light
    styles:
      card:
        - border-radius: 12px
        - height: 55px
        - padding: 4px
        - margin: 2px
      name:
        - font-size: 12px
        - font-weight: bold
        - padding-top: 4px
        - color: "#FFFFFF"
      icon:
        - width: 30px
        - color: "#43749f"
    state:
      - value: "off"
        styles:
          card:
            - background-color: transparent
          icon:
            - color: "#43749f"
          name:
            - color: "#43749f"
      - value: "on"
        color: "#D32F2F"
        styles:
          card:
            - background-color: "#C62828"
            - animation: pulse 1s infinite
          name:
            - color: "#FFFFFF"
          icon:
            - color: "#FFFFFF"
```
1. Enabled:
![Screenshot 2024-12-10 001923](https://github.com/user-attachments/assets/e13c5f36-251b-4590-92bb-1059396461d0)
2. Disabled:
![Screenshot 2024-12-10 002041](https://github.com/user-attachments/assets/8dfec758-743c-4781-9e6a-c6a1eeb2db75)
## Notifications
### Session Start Notification
```
alias: EV Charging - Session Started Notification
description: Notify when EV charging session starts with comprehensive error handling
mode: single
triggers:
  - entity_id: sensor.evse_eveus_state
    to: Charging
    trigger: state
conditions:
  - condition: template
    value_template: |
      {% set entities = [
        'sensor.evse_eveus_counter_a_energy',
        'input_number.ev_battery_capacity',
        'sensor.ev_soc_percent',
        'input_number.target_soc',
        'sensor.evse_eveus_curmeas1'
      ] %}
      {% set all_available = true %}
      {% for entity in entities %}
        {% if states(entity) in ['unavailable', 'unknown', ''] %}
          {% set all_available = false %}
        {% endif %}
      {% endfor %}
      {{ 
        all_available and
        states('sensor.evse_eveus_counter_a_energy')|float(0) >= 0 and
        states('input_number.ev_battery_capacity')|float(0) > 0
      }}
actions:
  - delay: "00:01:00"
  - data:
      title: "*EV Charging* 🪫 *Session Started*"
      message: >
        {% set current_soc = states('sensor.ev_soc_percent')|float(0) %}
        {% set battery_capacity = states('input_number.ev_battery_capacity')|float(0) %}
        {% set target_soc = states('input_number.target_soc')|float(0) %}
        {% set energy_needed = (target_soc - current_soc) * battery_capacity / 100 %}
        {% set time_to_target = states('sensor.evse_time_to_target_soc') %}
        {% set soc_delta = target_soc - current_soc %}
        {% set soc_delta_display = '+' + soc_delta|string if soc_delta > 0 else soc_delta|string %}
        {% set target_energy = states('sensor.evse_eveus_counter_a_energy')|float(0) + energy_needed %}
        {% set charging_current = states('sensor.evse_eveus_curmeas1')|float(0) %}
        {% set hours = time_to_target.split('h')[0]|int(0) if 'h' in time_to_target else 0 %}
        {% set minutes = time_to_target.split('h')[1].split('m')[0]|int(0) if 'h' in time_to_target else time_to_target.split('m')[0]|int(0) %}
        {% set completion_time = now() + timedelta(hours=hours, minutes=minutes) %}
        
        🔋 SoC: {{ current_soc|round(0) }}% → {{ target_soc|round(0) }}% (+{{ soc_delta|round(0) }}%)
        ⚡ Energy: {{ states('sensor.evse_eveus_counter_a_energy')|float(0)|round(0) }}kWh → {{ target_energy|round(0) }}kWh (+{{ energy_needed|round(0) }}kWh)
        🔌 Current: {{ charging_current|round(0) }}A
        ⏰ ETA: {{ completion_time.strftime('%H:%M %d.%m.%Y') }} (in {{ time_to_target }})
    action: notify.<NOTIFICATION_SERVICE_NAME>
max_exceeded: silent
```
### Current Change Notification
```
alias: EV Charging - Current Changed Notification
description: Notify when charging current changes during active charging
mode: single
triggers:
  - platform: state
    entity_id: sensor.evse_eveus_currentset
conditions:
  - condition: template
    value_template: >
      {% set entities = [
        'sensor.ev_soc_percent',
        'input_number.target_soc',
        'sensor.evse_time_to_target_soc',
        'sensor.evse_eveus_currentset',
        'sensor.evse_eveus'
      ] %}
      {% set all_available = true %}
      {% for entity in entities %}
        {% if states(entity) in ['unavailable', 'unknown', ''] %}
          {% set all_available = false %}
        {% endif %}
      {% endfor %}
      {% set state_num = state_attr('sensor.evse_eveus', 'state')|int(0) %}
      {{ 
        all_available and
        trigger.from_state.state != trigger.to_state.state and
        state_num == 4  # 4 is Charging state
      }}
actions:
  - delay: "00:01:00"
  - data:
      title: "*EV Charging* 🔌 *Current Changed*"
      message: >
        {% set current_soc = states('sensor.ev_soc_percent')|float(0) %}
        {% set target_soc = states('input_number.target_soc')|float(0) %}
        {% set soc_delta = target_soc - current_soc %}
        {% set time_to_target = states('sensor.evse_time_to_target_soc') %}
        {% if time_to_target not in ['unknown', '', 'unavailable', 'Not charging', 'Target reached'] %}
          {% set hours = time_to_target.split('h')[0]|int(0) if 'h' in time_to_target else 0 %}
          {% set minutes = time_to_target.split('h')[1].split('m')[0]|int(0) if 'h' in time_to_target else time_to_target.split('m')[0]|int(0) %}
          {% set completion_time = now() + timedelta(hours=hours, minutes=minutes) %}
          {% set eta_message = completion_time.strftime('%H:%M %d.%m.%Y') + ' (in ' + time_to_target + ')' %}
        {% else %}
          {% set eta_message = time_to_target %}
        {% endif %}
        
        🔌 Current: {{ states('sensor.evse_eveus_currentset')|float(0)|round(0) }}A
        🔋 SoC: {{ current_soc|round(0) }}% → {{ target_soc|round(0) }}% (+{{ soc_delta|round(0) }}%)
        ⏰ ETA: {{ eta_message }}
    action: notify.<NOTIFICATION_SERVICE_NAME>
max_exceeded: silent
```
### Session Complete Notification
Create an automation in Home Assistant by using the code below. Replace <NOTIFICATION_SERVICE_NAME> with your notification service name
```
alias: EV Charging - Session Completed Notification
description: Notify when EV charging session is complete with comprehensive error handling
mode: single
triggers:
  - entity_id: sensor.evse_eveus_state
    from: Charging
    trigger: state
conditions:
  - condition: template
    value_template: |
      {% set entities = [
        'sensor.evse_eveus_counter_a_energy',
        'input_number.ev_battery_capacity',
        'sensor.ev_soc_percent',
        'input_number.initial_ev_soc',
        'sensor.evse_eveus_newsessiontime',
        'sensor.evse_eveus_counter_a_cost'
      ] %}
      {% set all_available = true %}
      {% for entity in entities %}
        {% if states(entity) in ['unavailable', 'unknown', ''] %}
          {% set all_available = false %}
        {% endif %}
      {% endfor %}
      {{ 
        all_available and
        trigger.from_state.state == 'Charging' and 
        states('sensor.evse_eveus_counter_a_energy')|float(0) > 0 and
        states('input_number.ev_battery_capacity')|float(0) > 0
      }}
actions:
  - data:
      title: "*EV Charging* 🔋 *Session Completed*"
      message: >
        {% set session_time = states('sensor.evse_eveus_newsessiontime') %}
        {% set session_energy = states('sensor.evse_eveus_counter_a_energy')|float(0) %}
        {% set session_cost = states('sensor.evse_eveus_counter_a_cost')|float(0) %}
        {% set initial_soc = states('input_number.initial_ev_soc')|float(0) %}
        {% set final_soc = states('sensor.ev_soc_percent')|float(0) %}
        {% set battery_capacity = states('input_number.ev_battery_capacity')|float(0) %}
        {% set battery_added = (final_soc - initial_soc) * battery_capacity / 100 %}
        {% set soc_increase = final_soc - initial_soc %}
        {% set energy_delta = session_energy - battery_added %}
        
        🕒 Duration: {{ session_time }}
        🔋 SoC: {{ initial_soc|round(0) }}% → {{ final_soc|round(0) }}% (+{{ soc_increase|round(0) }}%)
        ⚡ Energy: {{ battery_added|round(0) }}kWh → {{ session_energy|round(0) }}kWh (+{{ energy_delta|round(0) }}kWh)
        💸 Session Cost: {{ session_cost|round(0) }}₴
        
        {% if final_soc < initial_soc %}
        ⚠️ Warning: Final SoC is lower than initial SoC. Possible measurement error.
        {% endif %}
    action: notify.<NOTIFICATION_SERVICE_NAME>
max_exceeded: silent
```

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
