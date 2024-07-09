# Zigbee Client Example

This test code demonstrates how to configure a Zigbee end device (client) and use it as a fake temperature sensor to send measurements to a gateway.

## How does Zigbee work?

See the README for the Gateway Example

## How does this example work ? 

First of all, we must configure our device using the Zigbee configuration for a temperature sensor. We can set the minimum and maximum values (e.g., min=-10, max=80).

```c
/* Create customized temperature sensor endpoint */
esp_zb_temperature_sensor_cfg_t sensor_cfg = ESP_ZB_DEFAULT_TEMPERATURE_SENSOR_CONFIG();
/* Set (Min|Max)MeasuredValure */
sensor_cfg.temp_meas_cfg.min_value = zb_temperature_to_s16(ESP_TEMP_SENSOR_MIN_VALUE);
sensor_cfg.temp_meas_cfg.max_value = zb_temperature_to_s16(ESP_TEMP_SENSOR_MAX_VALUE);
```

Now, we must create an endpoint for our application that will be added to a list, in the same way as the gateway.

```c
static esp_zb_ep_list_t *custom_temperature_sensor_ep_create(uint8_t endpoint_id, esp_zb_temperature_sensor_cfg_t *temperature_sensor)
{
    esp_zb_ep_list_t *ep_list = esp_zb_ep_list_create();
    esp_zb_ep_list_add_ep(
        ep_list,
        custom_temperature_sensor_clusters_create(temperature_sensor),
        endpoint_id,
        ESP_ZB_AF_HA_PROFILE_ID,
        ESP_ZB_HA_TEMPERATURE_SENSOR_DEVICE_ID);
    return ep_list;
}
```

To create endpoint clusters, we will create a `basic` cluster that holds all endpoint identification data. In this example, we define the manufacturer name as `Espressif` and the model identifier as `esp32h2`.

```c
// Create a cluster list for register all endpoint clusters
esp_zb_cluster_list_t *cluster_list = esp_zb_zcl_cluster_list_create();
// Create the basic cluster
esp_zb_attribute_list_t *basic_cluster = esp_zb_basic_cluster_create(&(temperature_sensor->basic_cfg));
// Add 2 attributes to the basic cluster
ESP_ERROR_CHECK(esp_zb_basic_cluster_add_attr(basic_cluster, ESP_ZB_ZCL_ATTR_BASIC_MANUFACTURER_NAME_ID, MANUFACTURER_NAME));
ESP_ERROR_CHECK(esp_zb_basic_cluster_add_attr(basic_cluster, ESP_ZB_ZCL_ATTR_BASIC_MODEL_IDENTIFIER_ID, MODEL_IDENTIFIER));
// Add the basic cluster to the cluster list
ESP_ERROR_CHECK(esp_zb_cluster_list_add_basic_cluster(cluster_list, basic_cluster, ESP_ZB_ZCL_CLUSTER_SERVER_ROLE));
// Add some ways to identify clusters
ESP_ERROR_CHECK(esp_zb_cluster_list_add_identify_cluster(cluster_list, esp_zb_identify_cluster_create(&(temperature_sensor->identify_cfg)), ESP_ZB_ZCL_CLUSTER_SERVER_ROLE));
ESP_ERROR_CHECK(esp_zb_cluster_list_add_identify_cluster(cluster_list, esp_zb_zcl_attr_list_create(ESP_ZB_ZCL_CLUSTER_ID_IDENTIFY), ESP_ZB_ZCL_CLUSTER_CLIENT_ROLE));
/* Add temperature measurement cluster for attribute reporting */
ESP_ERROR_CHECK(esp_zb_cluster_list_add_temperature_meas_cluster(cluster_list, esp_zb_temperature_meas_cluster_create(&(temperature_sensor->temp_meas_cfg)), ESP_ZB_ZCL_CLUSTER_SERVER_ROLE));
```

After that, we can register this created endpoint with the end device and configure it to send reports to the gateway every 5 seconds.

```c
static void esp_zb_task(void *pvParameters){
  ...
  /* Register the device */
  esp_zb_device_register(esp_zb_sensor_ep);
  
   /* Config the reporting info  */
    esp_zb_zcl_reporting_info_t reporting_info = {
        .direction = ESP_ZB_ZCL_CMD_DIRECTION_TO_SRV,
        .ep = HA_ESP_SENSOR_ENDPOINT,
        .cluster_id = ESP_ZB_ZCL_CLUSTER_ID_TEMP_MEASUREMENT,
        .cluster_role = ESP_ZB_ZCL_CLUSTER_SERVER_ROLE,
        .dst.profile_id = ESP_ZB_AF_HA_PROFILE_ID,
        .u.send_info.min_interval = 1,
        .u.send_info.max_interval = 5,
        .u.send_info.def_min_interval = 1,
        .u.send_info.def_max_interval = 5,        
        .u.send_info.delta.u16 = 100,
        .attr_id = ESP_ZB_ZCL_ATTR_TEMP_MEASUREMENT_VALUE_ID,
        .manuf_code = ESP_ZB_ZCL_ATTR_NON_MANUFACTURER_SPECIFIC
    };
    esp_zb_zcl_update_reporting_info(&reporting_info);
    ...
}
```

We can set a default value that the client sends to the gateway when it starts.

```c
esp_app_temp_sensor_handler(42);  
```

In this example, we simulate a temperature sensor, so we need to update the temperature value. To do that, we will use a button to update the value with a random number between -10 and 40 degrees Celsius.

```c
// This will change the value of the current temperature attribute in the cluster
static void esp_app_temp_sensor_handler(float temperature)
{
    int16_t measured_value = zb_temperature_to_s16(temperature);    
    esp_zb_zcl_set_attribute_val(HA_ESP_SENSOR_ENDPOINT,
                                 ESP_ZB_ZCL_CLUSTER_ID_TEMP_MEASUREMENT, ESP_ZB_ZCL_CLUSTER_SERVER_ROLE,
                                 ESP_ZB_ZCL_ATTR_TEMP_MEASUREMENT_VALUE_ID, &measured_value, false);    
}

// This method is called when the button is pressed for update the temperature value
static void esp_app_buttons_handler(switch_func_pair_t *button_func_pair)
{
    if (button_func_pair->func == SWITCH_ONOFF_TOGGLE_CONTROL)
    {        
        int temp = rand() % (40 + 1 - (-10)) + (-10);
        esp_app_temp_sensor_handler(temp);     
    }
}

// This method is called to bind the previous method to the button click event
// It's called when the system has booted
static esp_err_t deferred_driver_init(void)
{
    ESP_RETURN_ON_FALSE(switch_driver_init(button_func_pair, PAIR_SIZE(button_func_pair), esp_app_buttons_handler), ESP_FAIL, TAG,
                        "Failed to initialize switch driver");
    return ESP_OK;
}

void esp_zb_app_signal_handler(esp_zb_app_signal_t *signal_struct)
{
    ...
    switch (sig_type)
    {
        ...
        case ESP_ZB_BDB_SIGNAL_DEVICE_REBOOT:
            if (err_status == ESP_OK)
            {
                ESP_LOGI(TAG, "Deferred driver initialization %s", deferred_driver_init() ? "failed" : "successful");
                ...
            }
            ...
    }
    ...
}
```

So, here is a schema of the Zigbee node we produced with those pieces of code.

* Node :
  * Endpoint (ID=10 / PROFIL_ID=260 / DEVICE_ID=770 (Temperature) )
    * Basic Cluster (ID=0x0000)
      * Attribute ID=4 : Manufacturer name
      * Attribute ID=5 : Model identifier    
    * Temperature Measure Cluster (ID=0x0402)
      * Default attributes


## Example Output

As you run the example, you will see the following log:

```
I (463) ESP_ZB_TEMP_SENSOR: Initialize Zigbee stack
I (2993) gpio: GPIO[9]| InputEn: 1| OutputEn: 0| OpenDrain: 0| Pullup: 1| Pulldown: 0| Intr:2 
I (2993) ESP_ZB_TEMP_SENSOR: Deferred driver initialization successful
I (3003) ESP_ZB_TEMP_SENSOR: Device started up in non factory-reset mode
I (3003) ESP_ZB_TEMP_SENSOR: Device rebooted
I (3043) ESP_ZB_TEMP_SENSOR: ZDO signal: NLME Status Indication (0x32), status: ESP_OK
I (8053) ESP_ZB_TEMP_SENSOR: ZDO signal: NLME Status Indication (0x32), status: ESP_OK
I (13073) ESP_ZB_TEMP_SENSOR: ZDO signal: NLME Status Indication (0x32), status: ESP_OK
I (18103) ESP_ZB_TEMP_SENSOR: ZDO signal: NLME Status Indication (0x32), status: ESP_OK
```

This is a log when the device automatically sends a report to the server.
```
I (3043) ESP_ZB_TEMP_SENSOR: ZDO signal: NLME Status Indication (0x32), status: ESP_OK
```

When you press the button, you will not see the log on the client, but you will see the update on the gateway.