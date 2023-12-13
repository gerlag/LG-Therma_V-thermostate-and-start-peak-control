# LG-Therma_V-thermostate-and-start-peak-control
HA script that controls room temperature when Therma_V is in water-based AI-mode. Includes also start peak control.

## PROOF OF CONCEPT

This script is meant to run as a thermostate for the LG therma V heast pump with an external room temperature sensor. The script runs on a standard Home Assistant instalation  (HA) at an RPI or Odroid system. 

contributors of 

No need to mount the (ugly) RMC in your living room. 

LG Thermo must run in AI mode. The heating xurve 'Stooklijn' in the RMC must be properly set up prior to running this script. 

Any HA-readable temperature sensor can be used. 

1. setpoint and room temperature
The setpoint of the room temperature can be entererd in HA or by using an external device that can communicate with HA. 

As an example, the current setpoint and room temperature  can be read from 'Toon', using the following code snippet: 

In configuration.yaml: 
``` YAML
# read out TOON
# ref: https://www.domoticaforum.eu/viewtopic.php?f=97&t=12810&p=94570&hilit=happ_thermstat#p94570
rest:
    resource: "http://192.168.002.002/happ_thermstat?action=getThermostatInfo"
    scan_interval: 10
    sensor: 
        - name: CurrentSetpoint
          value_template: '{{ value_json.currentSetpoint |float(0) * 0.01 + 5}}'
          # Note: set setpoint at Toon 5 K lower. Example: sp 15  is meant to be sp 20 in HA.  If setpoint at Toon > 19 then ThermoV must be switched off and Gas boiler is in duty  
          unit_of_measurement: °C
        
        - name: CurrentTemperature
          value_template: '{{ value_json.currentTemp |float(0) * 0.01 }}' 
          unit_of_measurement: °C
```

2. PID contoller

The PID controller is based on the in HA built-in platform integration and platform derivative

In configuration.yaml:
```YAML
sensor:
    - platform: derivative
      source: sensor.actualtemp_vs_settemp
      name: derivative_actualtemp_vs_settemp
      round: 8
      unit_time: h 
      time_window: "00:5:00"  # we average over the last 5 minutes

    - platform: integration
      unique_id: b3bb2e79-f9e0-4d7d-8291-302e61c9fa2
      # source: sensor.actualtemp_vs_settemp
      source: input_number.windup_limited_diff_to_integrator
      name: integration_actualtemper_vs_settemp
      method: left
      unit_time: h 
      round: 8
```
In order to cope with the potentiaal huge value of the integration we store this value in an input_number so we can use the relative distance from this value as the actuel integrator value


In configuration.yaml:
```YAML
input_number:
  integrator_center_value: 
    name: Integrator center value
    min: -1e99
    max: 1e99
    unit_of_measurement: Kh
```
Further, we need to prevent winding up the integrator from the moment the output of the controller reaches its limit. In AI mode of the ThermaV the output is limited to + and - 5K 
The input difference of the integrator is kept at zero from the moment of limiting. This causes the output of the integrator not to increase or decrease anymore. 
An input_number will be used for this. We set this input_number to setpoint - measuredValue, or to zero in case the controller reaches its limit.

In configuration.yaml:
```YAML
input_number:
  windup_limited_diff_to_integrator:
    name: windup_limited_diff_actualtemp_versus_settemp
    min: -5
    max: 5
    unit_of_measurement: K 
```
The PID controller itself is created as an automation. 
Every miunute, the proportional, integration and derivative values are summed up send to the Therma_V 
The same automation also incorporates start peak control and tries to limit the aggressive powering up behavior of the ThermaV. 


In configuration.yaml:
```YAML
## needed to keep hand crafted automations untouched by GUI
homeassistant:
  packages: !include_dir_named packages 
## All of my regular automations go here
automation yaml: !include_dir_merge_list automations/
```
In automations/warmtepomp.yaml:
```YAML

```


   
