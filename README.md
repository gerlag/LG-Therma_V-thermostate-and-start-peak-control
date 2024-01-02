# LG-Therma_V-thermostate-and-start-peak-control
A HA script that controls the room temperature, using my Therma_V heat pump (9kW U44) as the ; heat source. The script includes also start peak and 'oil return peak' of the LG-Therma_V. 

## PROOF OF CONCEPT

[Credits: many contributors of the tweakers topic on https://gathering.tweakers.net/forum/list_messages/2017448/0]  

This script is meant to be used as a thermostate for the LG therma_V heat pump. It's using an external room temperature sensor. Any HA-readable temperature sensor can be used for measuring the room temperature. 

The script is being used on a standard Home Assistant instalation  (HA) at an RPI or Odroid system. 

It's NOT a 'copy and paste' script. Use it (partly) in your own Therma_V modbus script for inspiration. Check the sensor names to which this code is referring to. 

No need to mount the (ugly) RMC in your living room. In my case I wanted to keep the 'Toon' (https://www.eneco.nl/energieproducten/toon-thermostaat/)

Note: the LG Therma_V must be run in AI mode. The heating curve ('Stooklijn') in the RMC must be properly set up first. 

Below some 'frozen' code snippets. Actual updates or modifications, if any, can be found in de files section. 

1. setpoint and room temperature
The setpoint of the room temperature can be entererd in HA or by using an external device that can communicate with HA. 

As an example, the current setpoint and room temperature  can be read from my 'Toon', using the following code snippet: 

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

2. PID controller

The PID controller is based on the in HA built-in platforms integration and derivative

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
In order to cope with the potentiaal huge value of the integrationm  which can not be reset in HA, we store this value in an input_number so we can use the relative distance from that calue as the actuel integrator value. 

In configuration.yaml:
```YAML
input_number:
  integrator_center_value: 
    name: Integrator center value
    min: -1e99
    max: 1e99
    unit_of_measurement: Kh
```
Further, we need to prevent winding up the integrator from the moment the output of the controller reaches its limit, in our case this limit is the maximum + en - 5K shift of the actual value of the heating curve.  
The input difference of the integrator is kept at zero from the moment the maximum value has been reached. This causes the output of the integrator to not increase or decrease anymore. 
In HA, an input_number will be used for this. In normal operation, we set this input_number to ```setpoint - measuredValue```, in case the controller reaches its + or - 5K limit, we set the value to zero.

In configuration.yaml:
```YAML
input_number:
  windup_limited_diff_to_integrator:
    name: windup_limited_diff_actualtemp_versus_settemp
    min: -5
    max: 5
    unit_of_measurement: K 
```
In HA the output of the differentior, the derivative, does not go to zero if the input value is not changing. It keeps the last value before that moment. I think it's a bug, the developer thinks it's not a bug. Anyway, the derivative value should be zero in the case the input is not changing. If not, the D funtion of the controller creates a non zero output value, which of coursde is not wanted. The circumvent this problem, we add a very small amount of noise on top of the input value of the diffentiator. By doing this, the diffentiator still sees always a changing value, although very,very small, so it updates its output value every cycle and produces a zero output with some minimal noise on it. 

In In configuration.yaml:
```YAML
 ### noise difference between setpount and actual value  to keep derivative actualtemp_vs_settemp moving when zero
          noisy_actualtemp_vs_settemp:
              unit_of_measurement: K
              value_template: >
                {% set nd = states('sensor.actualtemp_vs_settemp') | float(0) %}
                {% set nd = nd+ (range(-51, 51) | random)/10000 %} 
                {{ nd }}  
```
The PID controller itself is created as an automation. 
Every minute, the proportional, integration and derivative values are summed up, the result, being the desired offset of the actual stooklijn temperature, is then send to the Therma_V. 
The same automation also incorporates start peak and oil return peak control. It tries also to limit the aggressive powering up behaviour of the ThermaV by arificailly keeping the diffence between setpoint and actual value low. 

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
################################################################################################################## 
# (1) room temperature control using  setpoint (sp) en measured value (mv) van Toon (external device of Eneco)   #
#   - set kp to a value thast workt well in your set up. In my case I use 3 as the default value.                #
#   - derivative is expermimental. Set Kd to 0 to disable the D component                                        #
#   - reader: https://www.cds.caltech.edu/~murray/courses/cds101/fa02/caltech/astrom-ch6.pdf                     #
# (2) Start peak overshootcontrol included (set overdeheuvelhulp to true or false)                               #
# (3) limit setpoint in order to limit too aggressive powering up of the comrpessor                              #
# (4) (re)set integrator 'zero' line                                                                             #
# (5) winding up limiter integrator                                                                              #  
#                                                                                                                #
# IMPORTANT: obligatory LG RMC settings:                                                                         #
#   - AI based water temperature control (output) *If input control is wanted, change 'o' under (2) accordingly  #
#   - 'heating curve (stooklijn)  MUST be set up first. MUST be adequate for YOUR system                         #
#   - hysteresis +4  and -1.5 appears to be acceptable in my set up.                                             #
#  -  Set input_number.max_delta_t in config.yaml to same number (or same-1) as the +hysteris  in RMC            #
##################################################################################################################
- id: '1676837085075'
  alias: Temp control using Toon sensor
  description: Temp control using Toon sensor
  trigger:
  - platform: time_pattern
    minutes: /01
  condition: []
  action:
  - service: modbus.write_register
    data:
      hub: lg_modbus
      slave: 1  
      address: 4  
      value: >
        {###################### (1) controller part PI(D) #########################################}
        {# set sp = states('sensor.currentSetpoint') | float(0)  #}  
        {# set mv = states('sensor.currentTemperature') | float(0)  #}
        {% set mvminsp = states('sensor.actualtemp_vs_settemp') | float(0)  %}
        {% set epwr =  states('sensor.electrical_power') | float(0) %} 
        {% if states('input_number.integrator_center_value') != 'unavailable' %}
           {% set integrationCenterValue = states('input_number.integrator_center_value')|float(0)  %}
           {% set integrator_value_running = states('sensor.integration_actualtemper_vs_settemp')|float(0)  %}
           {% set i = integrator_value_running - integrationCenterValue     %} 
           {% set d = states('sensor.derivative_actualtemp_vs_settemp')|float(0)    %} 
        {% else  %}
            {% set i = 0 | float(0) %}
            {% set d = 0 | float(0) %}
        {% endif %}
        {% set kp = states('input_number.kp') | float(0)  %}       
        {% set ki = states('input_number.ki') | float(0)  %}    
        {% set kd = states('input_number.kd') | float(0)  %}           
        {% set outputP = (kp*mvminsp)|float(0) %} 
        {% set outputI = (ki*i)|float(0) %} 
        {% set outputD = (kd*d)|float(0) %} 
        {% set of = -1*( outputP  + outputI + outputD )| round(3) %} 
        
        {###################### (1b) experiment anti oscillation #########################################}
        {% if true %}
             {# experiment anti oscillation #}
             {% set d_invertor = states('sensor.derivative_inverter')| float(0)   %}
             {% set k_anti_oscillation = states('input_number.k_anti_oscillation') | float(0)   %}
             {% set of = (of + d_invertor*k_anti_oscillation) | float(0) | round(3)  %}
        {% endif %}
        
        {###################### (2) overdeheuvelhulp startpiek #########################################} 
        {# set overdeheuvelhulp to false if not wanted #}
        {% set overdeheuvelhulp = true %} 
        {% if (overdeheuvelhulp) %}
            {% set maxDeltaT = states('input_number.max_delta_t') | float(0)  %} 
               {# set o = states('sensor.water_inlet_temp')| float(0)   #}
               {% set o = states('sensor.water_outlet_temp')| float(0)   %}
               {% set t = states('sensor.target_temp_circuit1') | float(0) %}
               {% set ofactual = states('sensor.shift_value_in_auto_mode_circuit1') | float(0)  %}            
            {% if ((states('binary_sensor.compressor_status') == "on" ) and ( epwr > 0.350 ) ) %}
               {% set allowedT = (o-maxDeltaT)|float(0) %}
               {% if  t >=  allowedT   %}
                 {% set hotter = ( (ofactual-of) + 0.8*(allowedT-t)  )  | round(0) | int %} 
               {% else %}
                 {% set hotter = ( (ofactual-of) + (allowedT-t)  )  | round(0) | int %} 
               {% endif %}
               {% if  hotter < 0   %}
                   {% set hotter = 0|int %} 
               {% endif %}
            {% else %}     
               {# WP is idle #}
               {% set hotter = 0 | int  %}
            {% endif %}
            {% set newof = (of+hotter) | float(0) | round(0) | int %}   
        {% else %}
           {% set newof = of | float(0) | round(0) | int %}
        {% endif %} 
        


        {############## (3) prevention of large power steps,              #########}
        {############## by limiting  setpoint to max 1 K above water_temp #########}
        {# set largePowerStepPrevention to false if not wanted #}
        {% set largePowerStepPrevention = true %}         
        {% if largePowerStepPrevention %}
            {% set presentstooklijnT = (t - ofactual) %}
            {% if  ( (t -o) > 1 )  or ( (presentstooklijnT + newof  -o ) > 1 ) %}
               {% set limitedTarget =  ( o +1  )     %}
               {% set newof= (limitedTarget -  presentstooklijnT)|round(0)|int %}
            {% endif %}
        {% endif %}

        {#### prepare number and send new offset to LG heatpump ####}
        {# limit new offset to plus and minus 5 K #}
        {% if (newof < -5) %}   
           {% set newof = -5  %}   
        {% elif (newof > 5) %}   
           {% set newof = 5  %}   
        {% endif %}
        {# process negative numbers correctly #}         
        {% if (newof < 0) %}
            {# compute the 2s complement #} 
            {% set newof = newof + 65535 +1 %}   
        {% endif %}  
        {{ newof | int  }}
        
    ################################################################################################################################        
    #  (4) (re)set integrator 'zero' line                                                                                          #
    #  use input_number to store the raw value of the integrator to be able to virtually (re)set the zero line of the I-action     #
    #  by substracting the stored value from the raw integrator data und using the result in the PI(D) controller as the zero line #
    ################################################################################################################################        
  - service: input_number.set_value
    data_template:
      entity_id: input_number.integrator_center_value
      value: >
          {% if states('input_number.integrator_center_value') != 'unavailable' %}
            {% set integrator_value_running = states('sensor.integration_actualtemper_vs_settemp')|float(0)  %}
            {% set integrationCenterValue = states('input_number.integrator_center_value')|float(0)    %} 
            {% set iDiff=  (integrator_value_running - integrationCenterValue)|float(0) %}
            {% set iDiffmax=  5|float(0) %}
            {% if (iDiff+2*iDiffmax) < iDiffmax  %}  
              {% set integrationCenterValue = integrator_value_running + 0*iDiffmax  %} 
              {{ integrationCenterValue }}
            {% elif (iDiff-2*iDiffmax) > iDiffmax   %}
              {% set integrationCenterValue =  integrator_value_running - 0*iDiffmax %} 
              {{ integrationCenterValue }}
            {% else %}
              {% set integrationCenterValue =  integrationCenterValue  %} 
              {{ integrationCenterValue}}
            {% endif %}
          {% endif %}

    ##################################################################################################
    #  (5) capping integrator windup: limit the building up of I action to max +/- 5K.               #
    ##################################################################################################
  - service: input_number.set_value
    data_template:
      entity_id: input_number.windup_limited_diff_to_integrator
      value: >
            {% set mvminsp = states('sensor.actualtemp_vs_settemp') | float(0)  %}
            {% set kp = states('input_number.kp') | float(0)  %}        
            {% set ki = states('input_number.ki') | float(0)  %}    
            {% set kd = states('input_number.kd') | float(0)  %}
            {% set integrationCenterValue = states('input_number.integrator_center_value')|float(0)  %}
            {% set integrator_value_running = states('sensor.integration_actualtemper_vs_settemp')|float(0)  %}
            {% set i = integrator_value_running - integrationCenterValue     %}
            {% set d = states('sensor.derivative_actualtemp_vs_settemp')|float(0)    %} 
            {% set outputP = (kp*mvminsp)|float(0) %} 
            {% set outputI = (ki*i)|float(0) %} 
            {% set outputD = (kd*d)|float(0) %} 
            {% set of = -1*( outputP  + outputI + outputD )| round(3) %} 
            {% if of|abs < 5 %}
              {{mvminsp}}
            {% else %}  
              {{ 0|float(0) }}
            {% endif %}    
            {# 
              Note: instead of recalculating the desired offset, it should be possible to test against the value of the offset register in the LG: states('sensor.shift_value_in_auto_mode_circuit1')    
            #}

    # force update of below sensors at each trigger. Needed to get proper zero values from derivative. (bypassing a bug (In my opinion) in HA )
      
  - service: homeassistant.update_entity
    target:
      entity_id: sensor.derivative_inverter
      
  - service: homeassistant.update_entity
    target:
      entity_id: sensor.noisy_actualtemp_vs_settemp  
      
  - service: homeassistant.update_entity
    target:
      entity_id: sensor.derivative_actualtemp_vs_settemp            
            
            
  mode: restart
```


   
