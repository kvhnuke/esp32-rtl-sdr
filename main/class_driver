/*
 * SPDX-FileCopyrightText: 2021-2022 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Unlicense OR CC0-1.0
 */

#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/semphr.h"
#include "esp_log.h"
#include "usb/usb_host.h"
#include "string.h"
#include "rtl-sdr.h"

#define CLIENT_NUM_EVENT_MSG 5

#define ACTION_OPEN_DEV 0x01
#define ACTION_GET_DEV_INFO 0x02
#define ACTION_GET_DEV_DESC 0x04
#define ACTION_GET_CONFIG_DESC 0x08
#define ACTION_GET_STR_DESC 0x10
#define ACTION_CLOSE_DEV 0x20
#define ACTION_EXIT 0x40

#define CTRL_OUT (USB_BM_REQUEST_TYPE_TYPE_VENDOR | USB_BM_REQUEST_TYPE_DIR_OUT)
#define CTRL_IN (USB_BM_REQUEST_TYPE_TYPE_VENDOR | USB_BM_REQUEST_TYPE_DIR_IN)

#define R820T_I2C_ADDR 0x34
#define R82XX_CHECK_ADDR 0x00
#define R82XX_CHECK_VAL 0x69

#define CTRL_TIMEOUT 300

#define R82XX_IF_FREQ 3570000

typedef struct
{
    usb_host_client_handle_t client_hdl;
    uint8_t dev_addr;
    usb_device_handle_t dev_hdl;
    uint32_t actions;
} class_driver_t;

typedef struct
{
    bool is_adsb;
    uint8_t *response_buf;
    bool is_done;
    usb_transfer_t *transfer;
} class_adsb_dev;

static const char *TAG = "CLASS";
static const char *TAG_ADSB = "ADSB";
static class_adsb_dev *adsbdev = NULL;
static rtlsdr_dev_t *rtldev = NULL;

static void
client_event_cb(const usb_host_client_event_msg_t *event_msg, void *arg)
{
    class_driver_t *driver_obj = (class_driver_t *)arg;
    switch (event_msg->event)
    {
    case USB_HOST_CLIENT_EVENT_NEW_DEV:
        if (driver_obj->dev_addr == 0)
        {
            driver_obj->dev_addr = event_msg->new_dev.address;
            rtlsdr_open(rtldev, event_msg->new_dev.address, driver_obj->client_hdl);
            // Open the device next
            // driver_obj->actions |= ACTION_OPEN_DEV;
        }
        break;
    case USB_HOST_CLIENT_EVENT_DEV_GONE:
        if (driver_obj->dev_hdl != NULL)
        {
            // Cancel any other actions and close the device next
            driver_obj->actions = ACTION_CLOSE_DEV;
        }
        break;
    default:
        // Should never occur
        abort();
    }
}

static void action_open_dev(class_driver_t *driver_obj)
{
    assert(driver_obj->dev_addr != 0);
    ESP_LOGI(TAG, "Opening device at address %d", driver_obj->dev_addr);
    ESP_ERROR_CHECK(usb_host_device_open(driver_obj->client_hdl, driver_obj->dev_addr, &driver_obj->dev_hdl));
    // Get the device's information next
    driver_obj->actions &= ~ACTION_OPEN_DEV;
    driver_obj->actions |= ACTION_GET_DEV_INFO;
}

static void action_get_info(class_driver_t *driver_obj)
{
    assert(driver_obj->dev_hdl != NULL);
    ESP_LOGI(TAG, "Getting device information");
    usb_device_info_t dev_info;
    ESP_ERROR_CHECK(usb_host_device_info(driver_obj->dev_hdl, &dev_info));
    ESP_LOGI(TAG, "\t%s speed", (dev_info.speed == USB_SPEED_LOW) ? "Low" : "Full");
    ESP_LOGI(TAG, "\tbConfigurationValue %d", dev_info.bConfigurationValue);
    // Todo: Print string descriptors

    // Get the device descriptor next
    driver_obj->actions &= ~ACTION_GET_DEV_INFO;
    driver_obj->actions |= ACTION_GET_DEV_DESC;
}

static void action_get_dev_desc(class_driver_t *driver_obj)
{
    assert(driver_obj->dev_hdl != NULL);
    ESP_LOGI(TAG, "Getting device descriptor");
    const usb_device_desc_t *dev_desc;
    ESP_ERROR_CHECK(usb_host_get_device_descriptor(driver_obj->dev_hdl, &dev_desc));
    if (dev_desc->idProduct == 0x2838 && dev_desc->idVendor == 0x0bda)
    {
        ESP_LOGI(TAG_ADSB, "Found ADSB Device");
        adsbdev = calloc(1, sizeof(class_adsb_dev));
        adsbdev->is_adsb = true;
    }
    usb_print_device_descriptor(dev_desc);
    // Get the device's config descriptor next
    driver_obj->actions &= ~ACTION_GET_DEV_DESC;
    driver_obj->actions |= ACTION_GET_CONFIG_DESC;
}

static void action_get_config_desc(class_driver_t *driver_obj)
{
    assert(driver_obj->dev_hdl != NULL);
    ESP_LOGI(TAG, "Getting config descriptor");
    const usb_config_desc_t *config_desc;
    ESP_ERROR_CHECK(usb_host_get_active_config_descriptor(driver_obj->dev_hdl, &config_desc));
    usb_print_config_descriptor(config_desc, NULL);
    // Get the device's string descriptors next
    driver_obj->actions &= ~ACTION_GET_CONFIG_DESC;
    driver_obj->actions |= ACTION_GET_STR_DESC;
}
static void transfer_cb(usb_transfer_t *transfer)
{
    // This is function is called from within usb_host_client_handle_events(). Don't block and try to keep it short
    //  struct class_driver_control *class_driver_obj = (struct class_driver_control *)transfer->context;
    // printf("Transfer type %d %ld \n", transfer->actual_num_bytes, transfer->flags);
    // printf("Transfer status %d, actual number of bytes transferred %d, databuffer size %d\n", transfer->status, transfer->actual_num_bytes, transfer->data_buffer_size);
}

static void transfer_read_cb(usb_transfer_t *transfer)
{
    for (int i = 0; i < transfer->actual_num_bytes; i++)
    {
        adsbdev->response_buf[i] = transfer->data_buffer[i];
    }
    adsbdev->is_done = true;
    //   printf("Transfer:Read type %d %ld \n", transfer->actual_num_bytes, transfer->flags);
    //  printf("Transfer:Read status %d, actual number of bytes transferred %d, databuffer size %d, %d\n", transfer->status, transfer->actual_num_bytes, transfer->data_buffer[8], adsbdev->response_buf[8]);
}

#define USB_SETUP_PACKET_INIT_CONTROL(setup_pkt_ptr, bm_reqtype, b_request, w_value, w_index, w_length) ({ \
    (setup_pkt_ptr)->bmRequestType = bm_reqtype;                                                           \
    (setup_pkt_ptr)->bRequest = b_request;                                                                 \
    (setup_pkt_ptr)->wValue = w_value;                                                                     \
    (setup_pkt_ptr)->wIndex = w_index;                                                                     \
    (setup_pkt_ptr)->wLength = w_length;                                                                   \
})
enum usb_reg
{
    USB_SYSCTL = 0x2000,
    USB_CTRL = 0x2010,
    USB_STAT = 0x2014,
    USB_EPA_CFG = 0x2144,
    USB_EPA_CTL = 0x2148,
    USB_EPA_MAXPKT = 0x2158,
    USB_EPA_MAXPKT_2 = 0x215a,
    USB_EPA_FIFO_CFG = 0x2160,
};
enum sys_reg
{
    DEMOD_CTL = 0x3000,
    GPO = 0x3001,
    GPI = 0x3002,
    GPOE = 0x3003,
    GPD = 0x3004,
    SYSINTE = 0x3005,
    SYSINTS = 0x3006,
    GP_CFG0 = 0x3007,
    GP_CFG1 = 0x3008,
    SYSINTE_1 = 0x3009,
    SYSINTS_1 = 0x300a,
    DEMOD_CTL_1 = 0x300b,
    IR_SUSPEND = 0x300c,
};

bool libusb_control_transfer(class_driver_t *driver_obj, uint8_t bm_req_type, uint8_t b_request, uint16_t wValue, uint16_t wIndex, unsigned char *data, uint16_t wLength, unsigned int timeout)
{
    usb_host_transfer_free(adsbdev->transfer);
    free(adsbdev->response_buf);
    size_t sizePacket = sizeof(usb_setup_packet_t) + wLength;
    usb_host_transfer_alloc(sizePacket, 0, &adsbdev->transfer);
    USB_SETUP_PACKET_INIT_CONTROL((usb_setup_packet_t *)adsbdev->transfer->data_buffer, bm_req_type, b_request, wValue, wIndex, wLength);
    adsbdev->transfer->num_bytes = sizePacket;
    adsbdev->transfer->device_handle = driver_obj->dev_hdl;
    adsbdev->transfer->timeout_ms = timeout;
    adsbdev->transfer->context = (void *)&driver_obj;
    adsbdev->transfer->callback = transfer_read_cb;
    adsbdev->is_done = false;
    adsbdev->response_buf = calloc(sizePacket, sizeof(uint8_t));

    if (bm_req_type == CTRL_OUT)
    {
        for (uint8_t i = 0; i < wLength; i++)
        {
            adsbdev->transfer->data_buffer[sizeof(usb_setup_packet_t) + i] = data[i];
        }
    }
    esp_err_t r = usb_host_transfer_submit_control(driver_obj->client_hdl, adsbdev->transfer);
    if (r != ESP_OK)
    {
        ESP_LOGI(TAG_ADSB, "libusb_control_transfer failed with %d", r);
        return false;
    }

    while (!adsbdev->is_done)
    {
        usb_host_client_handle_events(driver_obj->client_hdl, portMAX_DELAY);
    }
    for (uint8_t i = 0; i < wLength; i++)
    {
        data[i] = adsbdev->response_buf[sizeof(usb_setup_packet_t) + i];
    }
    return true;
}
bool rtlsdr_read_array(class_driver_t *driver_obj, uint8_t block, uint16_t addr, uint8_t *array, uint8_t len)
{
    uint16_t index = (block << 8);
    unsigned char *data = calloc(len, sizeof(char));
    bool ret = libusb_control_transfer(driver_obj, CTRL_IN, 0, addr, index, data, len, CTRL_TIMEOUT);
    *array = data[0];
    return ret;
}

bool rtlsdr_write_array(class_driver_t *driver_obj, uint8_t block, uint16_t addr, uint8_t *array, uint8_t len)
{
    uint16_t index = (block << 8) | 0x10;
    unsigned char *data = calloc(len, sizeof(char));
    data[0] = *array;
    bool ret = libusb_control_transfer(driver_obj, CTRL_OUT, 0, addr, index, data, len, CTRL_TIMEOUT);
    free(data);
    return ret;
}

enum blocks
{
    DEMODB = 0,
    USBB = 1,
    SYSB = 2,
    TUNB = 3,
    ROMB = 4,
    IRB = 5,
    IICB = 6,
};

uint8_t rtlsdr_i2c_read_reg(class_driver_t *driver_obj, uint8_t i2c_addr, uint8_t reg)
{
    uint16_t addr = i2c_addr;
    uint8_t data = 0;

    rtlsdr_write_array(driver_obj, IICB, addr, &reg, 1);
    rtlsdr_read_array(driver_obj, IICB, addr, &data, 1);

    return data;
}

bool rtlsdr_write_reg(class_driver_t *driver_obj, uint8_t block, uint16_t addr, uint16_t val, uint8_t len)
{
    uint16_t index = (block << 8) | 0x10;
    unsigned char data[2];
    if (len == 1)
        data[0] = val & 0xff;
    else
        data[0] = val >> 8;
    data[1] = val & 0xff;
    return libusb_control_transfer(driver_obj, CTRL_OUT, 0, addr, index, data, len, CTRL_TIMEOUT);
}

uint16_t rtlsdr_demod_read_reg(class_driver_t *driver_obj, uint8_t page, uint16_t addr, uint8_t len)
{
    unsigned char data[2] = {0, 0};
    uint16_t index = page;
    uint16_t reg;
    addr = (addr << 8) | 0x20;
    libusb_control_transfer(driver_obj, CTRL_IN, 0, addr, index, data, len, CTRL_TIMEOUT);
    ESP_LOGI(TAG_ADSB, "rtlsdr_demod_read_reg val %d %d", data[0], data[1]);
    reg = (data[1] << 8) | data[0];
    return reg;
}

int rtlsdr_demod_write_reg(class_driver_t *driver_obj, uint8_t page, uint16_t addr, uint16_t val, uint8_t len)
{
    unsigned char data[2];
    uint16_t index = 0x10 | page;
    addr = (addr << 8) | 0x20;

    if (len == 1)
        data[0] = val & 0xff;
    else
        data[0] = val >> 8;

    data[1] = val & 0xff;

    libusb_control_transfer(driver_obj, CTRL_OUT, 0, addr, index, data, len, CTRL_TIMEOUT);

    // size_t sizePacket = sizeof(usb_setup_packet_t) + len;
    // usb_transfer_t *transfer;
    // usb_host_transfer_alloc(sizePacket, 0, &transfer);
    // USB_SETUP_PACKET_INIT_CONTROL((usb_setup_packet_t *)transfer->data_buffer, CTRL_OUT, 0, addr, index, len);
    // transfer->data_buffer[8] = data[0];
    // if (len == 2)
    //     transfer->data_buffer[9] = data[1];
    // transfer->num_bytes = sizePacket;
    // transfer->device_handle = driver_obj->dev_hdl;
    // transfer->callback = transfer_cb;
    // transfer->context = (void *)&driver_obj;
    // int r = usb_host_transfer_submit_control(driver_obj->client_hdl, transfer);
    // if (r != ESP_OK)
    // {
    //     ESP_LOGI(TAG_ADSB, "rtlsdr_write_reg failed with %d", r);
    //     return false;
    // }
    int r = rtlsdr_demod_read_reg(driver_obj, 0x0a, 0x01, 1);
    ESP_LOGI(TAG_ADSB, "rtlsdr_demod_write_reg val %d", r);
    return (r == len) ? 0 : -1;
}

static const int fir_default[16] = {
    -54, -36, -41, -40, -32, -14, 14, 53,  /* 8 bit signed */
    101, 156, 215, 273, 327, 372, 404, 421 /* 12 bit signed */
};

int rtlsdr_set_fir(class_driver_t *driver_obj)
{
    uint8_t fir[20];

    int i;
    /* format: int8_t[8] */
    for (i = 0; i < 8; ++i)
    {
        const int val = fir_default[i];
        if (val < -128 || val > 127)
        {
            return -1;
        }
        fir[i] = val;
    }
    /* format: int12_t[8] */
    for (i = 0; i < 8; i += 2)
    {
        const int val0 = fir_default[8 + i];
        const int val1 = fir_default[8 + i + 1];
        if (val0 < -2048 || val0 > 2047 || val1 < -2048 || val1 > 2047)
        {
            return -1;
        }
        fir[8 + i * 3 / 2] = val0 >> 4;
        fir[8 + i * 3 / 2 + 1] = (val0 << 4) | ((val1 >> 8) & 0x0f);
        fir[8 + i * 3 / 2 + 2] = val1;
    }

    for (i = 0; i < (int)sizeof(fir); i++)
    {
        if (rtlsdr_demod_write_reg(driver_obj, 1, 0x1c + i, fir[i], 1))
            return -1;
    }

    return 0;
}

void rtlsdr_init_baseband(class_driver_t *driver_obj)
{
    /* initialize USB */
    rtlsdr_write_reg(driver_obj, USBB, USB_SYSCTL, 0x09, 1);
    rtlsdr_write_reg(driver_obj, USBB, USB_EPA_MAXPKT, 0x0002, 2);
    rtlsdr_write_reg(driver_obj, USBB, USB_EPA_CTL, 0x1002, 2);

    /* poweron demod */
    rtlsdr_write_reg(driver_obj, SYSB, DEMOD_CTL_1, 0x22, 1);
    rtlsdr_write_reg(driver_obj, SYSB, DEMOD_CTL, 0xe8, 1);

    /* reset demod (bit 3, soft_rst) */
    rtlsdr_demod_write_reg(driver_obj, 1, 0x01, 0x14, 1);
    rtlsdr_demod_write_reg(driver_obj, 1, 0x01, 0x10, 1);

    /* disable spectrum inversion and adjacent channel rejection */
    rtlsdr_demod_write_reg(driver_obj, 1, 0x15, 0x00, 1);
    rtlsdr_demod_write_reg(driver_obj, 1, 0x16, 0x0000, 2);

    /* clear both DDC shift and IF frequency registers  */
    for (int i = 0; i < 6; i++)
        rtlsdr_demod_write_reg(driver_obj, 1, 0x16 + i, 0x00, 1);

    rtlsdr_set_fir(driver_obj);

    /* enable SDR mode, disable DAGC (bit 5) */
    rtlsdr_demod_write_reg(driver_obj, 0, 0x19, 0x05, 1);

    /* init FSM state-holding register */
    rtlsdr_demod_write_reg(driver_obj, 1, 0x93, 0xf0, 1);
    rtlsdr_demod_write_reg(driver_obj, 1, 0x94, 0x0f, 1);

    /* disable AGC (en_dagc, bit 0) (this seems to have no effect) */
    rtlsdr_demod_write_reg(driver_obj, 1, 0x11, 0x00, 1);

    /* disable RF and IF AGC loop */
    rtlsdr_demod_write_reg(driver_obj, 1, 0x04, 0x00, 1);

    /* disable PID filter (enable_PID = 0) */
    rtlsdr_demod_write_reg(driver_obj, 0, 0x61, 0x60, 1);

    /* opt_adc_iq = 0, default ADC_I/ADC_Q datapath */
    rtlsdr_demod_write_reg(driver_obj, 0, 0x06, 0x80, 1);

    /* Enable Zero-IF mode (en_bbin bit), DC cancellation (en_dc_est),
     * IQ estimation/compensation (en_iq_comp, en_iq_est) */
    rtlsdr_demod_write_reg(driver_obj, 1, 0xb1, 0x1b, 1);

    /* disable 4.096 MHz clock output on pin TP_CK0 */
    rtlsdr_demod_write_reg(driver_obj, 0, 0x0d, 0x83, 1);
}

void rtlsdr_set_i2c_repeater(class_driver_t *driver_obj, int on)
{
    rtlsdr_demod_write_reg(driver_obj, 1, 0x01, on ? 0x18 : 0x10, 1);
}

#define TWO_POW(n) ((double)(1ULL << (n)))
#define DEF_RTL_XTAL_FREQ 28800000

static int rtlsdr_set_if_freq(class_driver_t *driver_obj, uint32_t freq)
{
    uint32_t rtl_xtal;
    int32_t if_freq;
    uint8_t tmp;
    int r;

    if (!driver_obj)
        return -1;

    // /* read corrected clock value */
    // if (rtlsdr_get_xtal_freq(driver_obj, &rtl_xtal, NULL))
    //     return -2;

    if_freq = ((freq * TWO_POW(22)) / DEF_RTL_XTAL_FREQ) * (-1);

    tmp = (if_freq >> 16) & 0x3f;
    r = rtlsdr_demod_write_reg(driver_obj, 1, 0x19, tmp, 1);
    tmp = (if_freq >> 8) & 0xff;
    r |= rtlsdr_demod_write_reg(driver_obj, 1, 0x1a, tmp, 1);
    tmp = if_freq & 0xff;
    r |= rtlsdr_demod_write_reg(driver_obj, 1, 0x1b, tmp, 1);

    return r;
}

static void action_get_str_desc(class_driver_t *driver_obj)
{
    assert(driver_obj->dev_hdl != NULL);
    usb_device_info_t dev_info;
    ESP_ERROR_CHECK(usb_host_device_info(driver_obj->dev_hdl, &dev_info));
    if (dev_info.str_desc_manufacturer)
    {
        ESP_LOGI(TAG, "Getting Manufacturer string descriptor");
        usb_print_string_descriptor(dev_info.str_desc_manufacturer);
    }
    if (dev_info.str_desc_product)
    {
        ESP_LOGI(TAG, "Getting Product string descriptor");
        usb_print_string_descriptor(dev_info.str_desc_product);
    }
    if (dev_info.str_desc_serial_num)
    {
        ESP_LOGI(TAG, "Getting Serial Number string descriptor");
        usb_print_string_descriptor(dev_info.str_desc_serial_num);
    }
    if (adsbdev != NULL && adsbdev->is_adsb)
    {
        ESP_LOGI(TAG_ADSB, "Doing ADSB Things");
        ESP_ERROR_CHECK(usb_host_interface_claim(driver_obj->client_hdl, driver_obj->dev_hdl, 1, 0));
        // dummy request
        rtlsdr_write_reg(driver_obj, 1, USB_SYSCTL, 0x09, 1);
        rtlsdr_init_baseband(driver_obj);
        rtlsdr_set_i2c_repeater(driver_obj, 1);
        uint8_t reg = rtlsdr_i2c_read_reg(driver_obj, R820T_I2C_ADDR, R82XX_CHECK_ADDR);
        if (reg == R82XX_CHECK_VAL)
        {
            fprintf(stderr, "Found Rafael Micro R820T tuner\n");
            rtlsdr_demod_write_reg(driver_obj, 1, 0xb1, 0x1a, 1);
            rtlsdr_demod_write_reg(driver_obj, 0, 0x08, 0x4d, 1);
            rtlsdr_set_if_freq(driver_obj, R82XX_IF_FREQ);
            rtlsdr_demod_write_reg(driver_obj, 1, 0x15, 0x01, 1);
        }
    }
    // Nothing to do until the device disconnects
    driver_obj->actions &= ~ACTION_GET_STR_DESC;
}

static void aciton_close_dev(class_driver_t *driver_obj)
{
    ESP_ERROR_CHECK(usb_host_device_close(driver_obj->client_hdl, driver_obj->dev_hdl));
    driver_obj->dev_hdl = NULL;
    driver_obj->dev_addr = 0;
    // We need to exit the event handler loop
    driver_obj->actions &= ~ACTION_CLOSE_DEV;
    driver_obj->actions |= ACTION_EXIT;
}

void class_driver_task(void *arg)
{
    SemaphoreHandle_t signaling_sem = (SemaphoreHandle_t)arg;
    class_driver_t driver_obj = {0};

    // Wait until daemon task has installed USB Host Library
    xSemaphoreTake(signaling_sem, portMAX_DELAY);

    ESP_LOGI(TAG, "Registering Client");
    usb_host_client_config_t client_config = {
        .is_synchronous = false, // Synchronous clients currently not supported. Set this to false
        .max_num_event_msg = CLIENT_NUM_EVENT_MSG,
        .async = {
            .client_event_callback = client_event_cb,
            .callback_arg = (void *)&driver_obj,
        },
    };
    ESP_ERROR_CHECK(usb_host_client_register(&client_config, &driver_obj.client_hdl));

    while (1)
    {
        if (driver_obj.actions == 0)
        {
            ESP_LOGI(TAG, "here looking for events");
            usb_host_client_handle_events(driver_obj.client_hdl, portMAX_DELAY);
        }
        else
        {
            if (driver_obj.actions & ACTION_OPEN_DEV)
            {
                action_open_dev(&driver_obj);
            }
            if (driver_obj.actions & ACTION_GET_DEV_INFO)
            {
                action_get_info(&driver_obj);
            }
            if (driver_obj.actions & ACTION_GET_DEV_DESC)
            {
                action_get_dev_desc(&driver_obj);
            }
            if (driver_obj.actions & ACTION_GET_CONFIG_DESC)
            {
                action_get_config_desc(&driver_obj);
            }
            if (driver_obj.actions & ACTION_GET_STR_DESC)
            {
                action_get_str_desc(&driver_obj);
            }
            if (driver_obj.actions & ACTION_CLOSE_DEV)
            {
                aciton_close_dev(&driver_obj);
            }
            if (driver_obj.actions & ACTION_EXIT)
            {
                break;
            }
        }
    }

    ESP_LOGI(TAG, "Deregistering Client");
    ESP_ERROR_CHECK(usb_host_client_deregister(driver_obj.client_hdl));

    // Wait to be deleted
    xSemaphoreGive(signaling_sem);
    vTaskSuspend(NULL);
}