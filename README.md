# Solar versus Grid Charge
A Home Assistance guide to calculate Solar vs Grid EV charging and thereby estimating the refund of electricity taxes (in Denmark) 

This guide is mostly relevant for individuals located in Denmark, who have Looad (and perhaps upcoming NRGi) subscription and gets a partial refund of the elextricity taxes (Refusion af elafgift) for the amount of EV charging from the grid and not from solar.

![image](https://github.com/user-attachments/assets/d0482b7a-9780-4f34-ad2a-2d1825f84824)

# Prerequisites
- a sensor for the current Grid import/export power (in Watts) with update frequency of 10 seconds or faster (In this example sensor.grid_active_power)
- a sensor for the current EVSE power (in Watts) with update frequency of 10 seconds or faster (I have included a template sensor for this below, sensor.easee_watts)

# Disclamer
Please be aware, that we are dealing with template sensors and integral sensors which approximates the energy consumption, so the calculations might not be 100% correct, but it will give you an idea about the amount of energy charged from Solar and from Grid.

# Create Template sensors
Create the following sensors - The first one is an example of converting Easee Power in kW to W, and adding a 10 second update frequency, which is important for the following Integral Helper sensors described in next chapter

Add the following to your Template Sensor section:
```
- trigger:
    - platform: time_pattern
      seconds: /10
  sensor:      
# A Sensor for the EVSE power measured in Watts. In this example I convert Easee from kW to W.
  - name: Easee Watts
    unique_id: easee_watts
    unit_of_measurement: "W"
    device_class: power
    state: >-
      {% set easee_kw = states('sensor.easee_power') | round(3) %}
      {{ (easee_kw * 1000) | round }}
    attributes:
      update: "{{ now().second }}"
   

#Calculate how much charging from solar or battery
  - name: Charging from Solar or Battery Power
    unique_id: charging_from_solar_or_battery_power
    unit_of_measurement: "W"
    device_class: power
    state: >-
      {% set grid_power = states('sensor.grid_active_power') | round %}
      {% set charging_power = states('sensor.easee_watts') | round %}
      {% if charging_power > 50 and grid_power <= 0 %}
        {{ (charging_power) | round }}
      {% elif charging_power > 50 and (charging_power - (grid_power)) > 0 %}
        {{ (charging_power - grid_power) | round }}
      {% else %}
        {{ 0 }}
      {% endif %}
    attributes:
      update: "{{ now().second }}"


# Calculate how much charging from grid, these are the two Integral Helpers created in next step, so mind the naming of the sensors.
  - name: Charging Energy Grid (refusion)
    unique_id: charging_energy_grid_refusion
    unit_of_measurement: "kWh"
    device_class: energy
    state_class: total_increasing
    state: >-
      {% set energy_total = states('sensor.charging_energy_total_calculated') | float %}
      {% set energy_solar = states('sensor.charging_energy_solar_calculated') | float %}
      {{ ((energy_total - energy_solar)) | round(2) }} 
```

Note: The reason why we are adding attributes to the sensors is to force Home Assistant to update the following Integral helpers, even if the value stays the same (for example charging with 10kW for 2 hours, will provide wrong values in the Reimann Sum Integral if updated that infrequently)


# Create Helpers
Create two Reimann sum Integral helpers, with the following setup (Based on the two first sensors created above) :

A sensor to calculate the total charging energy
![image](https://github.com/user-attachments/assets/c36404b0-3409-48a2-ad52-61450744dfef)

A sensor to calculate the charging energy coming from solar/battery
![image](https://github.com/user-attachments/assets/0d4e577c-fba7-4b96-9ac7-b9ec28dc6b86)

Note 1: You can play around with "Trapezodial" or "Left" to see what gives you the most accurate results (depends on update requency etc.)
Note 2: The reason for creating the Charging Energy Total sensor is that Easee only provides sensor-updates for the total energy on a very infrequently basis, which gives inaccurate results. If you for example measure the power and energy to you car-charger with a Shelly 3PM you can just use that energy sensor instead of creating the Charging Energy Total sensor.

# Dashboards
Now you sould be able to create some dashboards, and add the sensors to the Home Assistant Energy Dashboard:

![image](https://github.com/user-attachments/assets/0518ece9-2dfd-42d9-ae8d-55ccfa5abf9f)

```
type: entities
entities:
  - entity: sensor.easee_watts
  - entity: sensor.charging_from_solar_or_battery_power
  - entity: sensor.charging_energy_total_calculated
  - entity: sensor.charging_energy_solar_calculated
  - entity: sensor.charging_energy_grid_refusion
```

```
type: custom:apexcharts-card
chart_type: pie
apex_config:
  legend:
    show: false
  dataLabels:
    enabled: true
    dropShadow:
      enabled: true
    formatter: |
      EVAL:function(value) {
        return Number.parseFloat(Number(value)).toFixed(1) + "%";
      }
  tooltip:
    enabled: false
    onDatasetHover:
      highlightDataSeries: false
  chart:
    height: 250px
  fill:
    type: gradient
    gradient:
      shadeIntensity: 0.1
      opacityFrom: 1
      opacityTo: 1
      inverseColors: true
      type: vertical
      stops:
        - 0
        - 80
        - 100
  plotOptions:
    bar:
      borderRadius: 10
      dataLabels:
        position: top
header:
  show: true
  show_states: true
  colorize_states: true
  title: Charging Source since 01-10-2024
all_series_config:
  show:
    legend_value: false
    datalabels: percent
    in_header: raw
  float_precision: 2
series:
  - entity: sensor.charging_energy_solar_calculated
    name: Solar
  - entity: sensor.charging_energy_grid_refusion
    name: Grid
update_interval: 1min
```

# Energy Dashboard
Add the two sensors as Individual devices to your Energy Dashboard:
![image](https://github.com/user-attachments/assets/4a838c3c-6a17-4da8-bc49-0af7c6dc5edb)

![image](https://github.com/user-attachments/assets/9201c413-48a1-4d30-9556-b9447e0d419f)

![image](https://github.com/user-attachments/assets/18911e2d-db9e-46bb-96e9-a22f001a9bcd)


And finally add the Charging Energy Grid sensor to your Cas Consumption like this:
![image](https://github.com/user-attachments/assets/4bdbd3d7-de3c-4cb9-a159-1030f470c179)

Then your Energy Sources will calculate the actual costs after refund of the taxes:
![image](https://github.com/user-attachments/assets/d990b0f8-0393-4226-aceb-425d43716ba6)






