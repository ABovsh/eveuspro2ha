# Eveus Pro integration with Home Assistant

# Disclaimer
The code within this repository comes with no guarantee, the use of this code is your responsibility.

I take NO responsibility and/or liability for how you choose to use information available here. By using any of the files available in this repository, you understand that you are AGREEING TO USE AT YOUR OWN RISK.

# Info

Replace the following attributes below in the code with your actual values:
1. http://<IP_Address>/main - specify your Eveus charger IP address. Example - http://192.168.1.1/main
2. <EVEUS_USER>
3. <EVEUS_PASSWORD>
4. <EV_BATTERY_CAPACITY_IN_KWH>

# Changelog comparing with original version
1. Added description for sensors
2. Corrected statuses mapping logic. Added sub-statuses mapping logic
3. Fixed current management
4. Fixed "Stop charging" logic
5. Added SOC sensors
6. Added Time to target sensor
7. Added Counter A and Counter B sensors

# Configuration steps
# 1. Add sensors to the /config/configuration.yaml file
```
#Sensors for Eveus car charger
- platform: rest
  name: EVSE Eveus
  resource: http://<IP_Address>/main
  username: <EVEUS_USER>
  password: <EVEUS_PASSWORD>
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
  scan_interval: 60  #Update every N seconds

- platform: template
  sensors:
    #Sensor for session duration formatted as time
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

    #Sensor for EVSE enabled status
    evse_eveus_enabled:
      value_template: "{{ state_attr('sensor.evse_eveus', 'evseEnabled') }}"
      friendly_name: "EVSE Enabled"

    #Sensor for EVSE state with human-readable description
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

    #Sensor for EVSE sub-state with human-readable description
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

    #Sensor for current set in amperes
    evse_eveus_currentset:
      value_template: "{{ state_attr('sensor.evse_eveus', 'currentSet') }}"
      friendly_name: "Current set"
      unit_of_measurement: "A"

    #Sensor for current designed in amperes
    evse_eveus_curdesign:
      value_template: "{{ state_attr('sensor.evse_eveus', 'curDesign') }}"
      unit_of_measurement: "A"
      friendly_name: "Current designed"

    #Sensor for voltage measurement 1
    evse_eveus_voltmeas1:
      value_template: "{{ state_attr('sensor.evse_eveus', 'voltMeas1') | round(1) }}"
      unit_of_measurement: "V"
      friendly_name: "Voltage"

    #Sensor for current measurement 1
    evse_eveus_curmeas1:
      value_template: "{{ state_attr('sensor.evse_eveus', 'curMeas1') | round(1) }}"
      unit_of_measurement: "A"
      friendly_name: "Charger Current"

    #Sensor for power measurement in watts
    evse_eveus_powermeas:
      value_template: "{{ state_attr('sensor.evse_eveus', 'powerMeas') | round(1) }}"
      unit_of_measurement: "W"
      friendly_name: "Charger Power"

    #Sensor for temperature of the box
    evse_eveus_temperature1:
      value_template: "{{ state_attr('sensor.evse_eveus', 'temperature1') }}"
      unit_of_measurement: "°C"
      friendly_name: "Temp of box"

    #Sensor for temperature of the plug
    evse_eveus_temperature2:
      value_template: "{{ state_attr('sensor.evse_eveus', 'temperature2') }}"
      unit_of_measurement: "°C"
      friendly_name: "Temp of plug"

    #Sensor for ground status
    evse_eveus_ground:
      value_template: "{% if state_attr('sensor.evse_eveus', 'ground') == 1 %}Yes{% else %}No{% endif %}"
      friendly_name: "Ground"

    #Sensor for ground control status
    evse_eveus_groundctrl:
      value_template: "{% if state_attr('sensor.evse_eveus', 'groundCtrl') == 2 %}Yes{% else %}No{% endif %}"
      friendly_name: "Ground control"

    #Sensor for adaptive mode voltage
    evse_eveus_aivoltage:
      value_template: "{{ state_attr('sensor.evse_eveus', 'aiVoltage') }}"
      unit_of_measurement: "V"
      friendly_name: "Adaptive mode voltage"

    #Sensor for adaptive mode current
    evse_eveus_aicurrent:
      value_template: "{{ state_attr('sensor.evse_eveus', 'aiModecurrent') }}"
      unit_of_measurement: "A"
      friendly_name: "Adaptive mode current"

    #Sensor for session energy
    evse_eveus_sessionenergy:
      value_template: "{{ (state_attr('sensor.evse_eveus', 'sessionEnergy') | float) | round(1) }}"
      unit_of_measurement: "kWh"
      friendly_name: "Session Energy"

    #Sensor for session time
    evse_eveus_sessiontime:
      value_template: "{{ state_attr('sensor.evse_eveus', 'sessionTime') }}"
      unit_of_measurement: "s"
      friendly_name: "Session Time"

    #Sensor for total energy
    evse_eveus_totalenergy:
      value_template: "{{ (state_attr('sensor.evse_eveus', 'totalEnergy') | float) | round(1) }}"
      unit_of_measurement: "kWh"
      friendly_name: "Total Energy"

    #Sensor for AI status
    evse_eveus_aistatus:
      value_template: "{{ state_attr('sensor.evse_eveus', 'aiStatus') }}"
      friendly_name: "Adaptive mode status"

    #Sensor for battery voltage
    evse_eveus_vbat:
      value_template: "{{ state_attr('sensor.evse_eveus', 'vBat') }}"
      unit_of_measurement: "V"
      friendly_name: "Battery voltage"

    #Sensor for system time
    evse_eveus_systemtime:
      value_template: "{{ state_attr('sensor.evse_eveus', 'systemTime') }}"
      friendly_name: "System Time"

    #Sensor for Counter A Energy
    evse_eveus_counter_a_energy:
      friendly_name: "Counter A Energy (kWh)"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM1') | round(2) }}"
      unit_of_measurement: "kWh"

    #Sensor for Counter A Cost
    evse_eveus_counter_a_cost:
      friendly_name: "Counter A Cost"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM1_money') | round(2) }}"
      unit_of_measurement: "₴"

    #Sensor for Counter B Energy
    evse_eveus_counter_b_energy:
      friendly_name: "Counter B Energy (kWh)"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM2') | round(2) }}"
      unit_of_measurement: "kWh"

    #Sensor for Counter B Cost
    evse_eveus_counter_b_cost:
      friendly_name: "Counter B Cost"
      value_template: "{{ state_attr('sensor.evse_eveus', 'IEM2_money') | round(2) }}"
      unit_of_measurement: "₴"

    #Sensor for EV SOC in kWh using Counter A and initial SOC
    ev_soc_kwh:
      friendly_name: "EV State of Charge (kWh)"
      value_template: >
        {% set initial_soc = states('input_number.initial_ev_soc') | float %}
        {% set max_capacity = <EV_BATTERY_CAPACITY_IN_KWH> %}
        {% set energy_charged = states('sensor.evse_eveus_counter_a_energy') | float %}
        {% set correction_factor = states('input_number.ev_soc_correction') | float / 100 %}
        {% if initial_soc >= 0 and energy_charged >= 0 %}
          {{ ((initial_soc / 100 * max_capacity) + energy_charged * (1 - correction_factor)) | round(2) }}
        {% else %}
          unknown
        {% endif %}
      unit_of_measurement: "kWh"

    #Sensor for EV SOC in percentage using Counter A and initial SOC
    ev_soc_percent:
      friendly_name: "EV State of Charge (%)"
      value_template: >
        {% set initial_soc = states('input_number.initial_ev_soc') | float %}
        {% set max_capacity = <EV_BATTERY_CAPACITY_IN_KWH> %}
        {% set energy_charged = states('sensor.evse_eveus_counter_a_energy') | float %}
        {% set correction_factor = states('input_number.ev_soc_correction') | float / 100 %}
        {% if initial_soc >= 0 and energy_charged >= 0 %}
          {% set total_soc = (initial_soc / 100 * max_capacity + energy_charged * (1 - correction_factor)) %}
          {{ ((total_soc / max_capacity) * 100) | round(2) }}
        {% else %}
          unknown
        {% endif %}
      unit_of_measurement: "%"

    #Sensor for Time to Target SOC considering losses
    evse_time_to_target_soc:
      friendly_name: "Time to Target SOC"
      value_template: >-
        {% set total_capacity = <EV_BATTERY_CAPACITY_IN_KWH> %}
        {% set current_soc = states('sensor.ev_soc_percent') | float(0) %}
        {% set initial_soc = states('input_number.initial_ev_soc') | float(0) %}
        {% set energy_charged = states('sensor.evse_eveus_counter_a_energy') | float(0) %}
        {% set correction_factor = states('input_number.ev_soc_correction') | float(0) / 100 %}
        {% set charging_power = states('sensor.evse_eveus_powermeas') | float(0) %}
        {% set target_soc = states('input_number.target_soc') | float(80) %}
        
        {% set target_capacity = total_capacity * target_soc / 100 %}
        {% set current_capacity = total_capacity * current_soc / 100 %}
        
        {% set remaining_capacity_kwh = target_capacity - current_capacity %}
        
        {% if current_soc >= target_soc %}
          SOC is already at or above target
        {% else %}
          {% if remaining_capacity_kwh <= 0 %}
            SOC is already above target
          {% else %}
            {% set effective_power = charging_power * (1 - correction_factor) / 1000 %}
            
            {% if charging_power == 0 %}
              Charging power is unavailable
            {% elif effective_power > 0 %}
              {% set time_seconds = (remaining_capacity_kwh * 3600) / effective_power %}
              {% set days = time_seconds // 86400 %}
              {% set hours = (time_seconds % 86400) // 3600 %}
              {% set minutes = (time_seconds % 3600) // 60 %}
              
              {% if time_seconds < 3600 %}
                {{ '%dm' % minutes }}
              {% else %}
                {{ '%dd ' % days if days else '' }}{{ '%dh ' % hours if hours else '' }}{{ '%dm' % minutes if minutes else '' }}
              {% endif %}
            {% else %}
              unknown
            {% endif %}
          {% endif %}
        {% endif %}
```
# 2. Add switches to the /config/configuration.yaml file
```
command_line:
  - switch:
      name: EVSE Stop Charging
      unique_id: evse_eveus_stop_charging
      command_on: 'curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST -H "Content-type: application/x-www-form-urlencoded" "http://<IP_Address>/pageEvent" -d "pageevent=evseEnabled&evseEnabled=1"'
      command_off: 'curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST -H "Content-type: application/x-www-form-urlencoded" "http://<IP_Address>/pageEvent" -d "pageevent=evseEnabled&evseEnabled=0"'
      command_state: 'curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST "http://<IP_Address>/main" | jq ".evseEnabled"'
      value_template: '{{ value == "1" }}'  #ON if evseEnabled is 1, OFF if evseEnabled is 0

  - switch:
      name: EVSE OneCharge
      unique_id: evse_eveus_one_charge
      command_on: 'curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST -H "Content-type: application/x-www-form-urlencoded" "http://<IP_Address>/pageEvent" -d "pageevent=oneCharge&oneCharge=1"'
      command_off: 'curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST -H "Content-type: application/x-www-form-urlencoded" "http://<IP_Address>/pageEvent" -d "pageevent=oneCharge&oneCharge=0"'
      command_state: 'curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST "http://<IP_Address>/main" | jq ".oneCharge"'
      value_template: '{{ value == "1" }}'  #ON if oneCharge is 1, OFF if oneCharge is 0
```
# 3. Add current regulator into /config/configuration.yaml
```
shell_command:
  evse_current_incr: "curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST -H 'Content-type: application/x-www-form-urlencoded' 'http://<IP_Address>/pageEvent' -d \"currentSet=$(($(curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST 'http://<IP_Address>/main' | jq '.currentSet')+1))\""
  evse_current_decr: "curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST -H 'Content-type: application/x-www-form-urlencoded' 'http://<IP_Address>/pageEvent' -d \"currentSet=$(($(curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST 'http://<IP_Address>/main' | jq '.currentSet')-1))\""
  evse_current_set: "curl -s -u <EVEUS_USER>:<EVEUS_PASSWORD> -X POST -H 'Content-type: application/x-www-form-urlencoded' 'http://<IP_Address>/pageEvent' -d \"currentSet={{ '%02d'|format(states('input_number.evse_eveus_current')|int) }}\""
```
# 4. Add input numbers into /config/configuration.yaml
```
input_number:
  initial_ev_soc:
    name: Initial EV State of Charge (%)
    min: 0
    max: 100
    step: 1
    mode: slider
    icon: mdi:BatteryCharging40

  ev_soc_correction:
    name: SOC Correction Factor (%)
    min: 5
    max: 10  #Adjust the max if needed
    step: 0.1
    initial: 7.5  #Default value
    mode: slider  #Optional
    icon: mdi:TuneVariant

  target_soc:
    name: Target SOC (%)
    min: 80
    max: 100
    step: 10
    initial: 80  #Default target SOC
    mode: slider  #Optional
    icon: mdi:Battery

  evse_eveus_current:
    name: Charger Set Current
    min: 8
    max: 32
    step: 1
    initial: 16  #Default value
    mode: slider  #Optional
    icon: mdi:current-dc
```
# 5. Validate configuration in Home Assistant
Configuration -> Server Controls
Press `CHECK CONFIGURATION` button in `Check Configuration` section

# 6. Restart HA server
Configuration -> Server Controls
Press `RESTART` in `Server managementn` section

# 7. Create an entity card in your Home Assistant dashboard
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
  - entity: sensor.evse_eveus_state
    name: Charger State
    icon: mdi:car-electric
  - entity: sensor.evse_eveus_substate
    name: Substate
    icon: mdi:car-cog
  - entity: input_number.evse_eveus_current
    name: Current (A)
    icon: mdi:current-ac
  - entity: input_number.initial_ev_soc
    name: Initial SOC (%)
    icon: mdi:calendar-clock
  - entity: input_number.target_soc
    name: Target SOC (%)
    icon: mdi:target
  - entity: sensor.evse_eveus_powermeas
    name: Power (W)
    icon: mdi:flash
  - entity: sensor.evse_eveus_counter_a_energy
    name: Session Energy (kWh)
    icon: mdi:counter
  - entity: switch.evse_one_charge
    name: One Charge Mode
    icon: mdi:power-plug
  - entity: switch.evse_stop_charging2
    name: Stop Charging 2
    icon: mdi:power-plug-off
  - entity: switch.evse_reset_counter_a
    name: Reset Energy Counter
    icon: mdi:reload
  - entity: input_number.ev_soc_correction
show_header_toggle: false
```
![Screenshot 2024-12-05 224217](https://github.com/user-attachments/assets/03e2573c-a313-4db9-9bb6-7870d10f39df)

# SOC CALCULATION
For proper SOC calculation it`s important to:
1. Set your EV Battery capacity (one time action)
2. Reset Counter A before every car charge
3. Set current SOC level before every car charge
4. Set the right corretion level (energy loses in % - 7.5 by default)
