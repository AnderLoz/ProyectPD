# Projecte PD: Pedal de guitarra
## **Introducció**
![Muntatge](muntatge.jpg)
* ESP32-S3
* DAC MAX98357A
* ADC PCM1808
* Connectors d'audio JACK de 6,35mm i 3,5mm

## **Software i el seu funcionament**

### Capçalera
A la capçalera hi trobem les llibreries necessaries per el codi que en aquest cas son només 2: Arduino.h i driver/i2s.h. També hi ha la configuració i l'assignació dels pins per el bus I2S, l'ADC i el DAC. També hi ha una funció que processa l'audio donant-lo l'efecte de overdrive. 


```cpp
// Configuración de pines y constantes
const i2s_port_t i2s_num_adc = I2S_NUM_0; // Utilizamos el puerto I2S 0 para ADC
const i2s_port_t i2s_num_dac = I2S_NUM_1; // Utilizamos el puerto I2S 1 para DAC

// Configuración del I2S para recibir datos del ADC externo
i2s_config_t i2s_config_adc = {
    .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX),
    .sample_rate = 16000,
    .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
    .channel_format = I2S_CHANNEL_FMT_ONLY_LEFT,
    .communication_format = I2S_COMM_FORMAT_I2S_MSB,
    .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1,
    .dma_buf_count = 8,
    .dma_buf_len = 64,
    .use_apll = false,
    .tx_desc_auto_clear = true,
    .fixed_mclk = 0
};

// Configuración del I2S para enviar datos al DAC externo
i2s_config_t i2s_config_dac = {
    .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX),
    .sample_rate = 16000,
    .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
    .channel_format = I2S_CHANNEL_FMT_ONLY_LEFT,
    .communication_format = I2S_COMM_FORMAT_I2S_MSB,
    .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1,
    .dma_buf_count = 8,
    .dma_buf_len = 64,
    .use_apll = false,
    .tx_desc_auto_clear = true,
    .fixed_mclk = 0
};

// Configuración de los pines I2S (ESP32-S3)
i2s_pin_config_t pin_config_adc = {
    .bck_io_num = 5,   // BCK (Bit Clock)
    .ws_io_num = 6,    // WS (Word Select)
    .data_out_num = I2S_PIN_NO_CHANGE, // Salida de datos no utilizada para ADC
    .data_in_num = 8   // Entrada de datos desde el ADC
};

i2s_pin_config_t pin_config_dac = {
    .bck_io_num = 5,   // BCK (Bit Clock)
    .ws_io_num = 6,    // WS (Word Select)
    .data_out_num = 7, // Salida de datos para el DAC
    .data_in_num = I2S_PIN_NO_CHANGE   // Entrada de datos no utilizada para DAC
};

// Función de procesamiento de overdrive
int16_t applyOverdrive(int16_t sample) {
    float gain = 2.0; // Ganancia de la señal de entrada
    float threshold = 0.8; // Umbral de corte
    float sample_f = sample / 32768.0 * gain;
    
    if (sample_f > threshold) {
        sample_f = threshold;
    } else if (sample_f < -threshold) {
        sample_f = -threshold;
    }

    // Normalizamos de vuelta al rango de 16 bits
    return (int16_t)(sample_f * 32768.0);
}

```

### **- Estructura del Setup**
En el setup inicialitzem els dos busos I2S tant per el ADC com per el DAC

```cpp
// Configuración inicial
void setup() {
    Serial.begin(115200);

    // Inicializar I2S para recibir datos del ADC
    if (i2s_driver_install(i2s_num_adc, &i2s_config_adc, 0, NULL) != ESP_OK) {
        Serial.println("Error instalando I2S para ADC");
        return;
    }
    if (i2s_set_pin(i2s_num_adc, &pin_config_adc) != ESP_OK) {
        Serial.println("Error configurando pines I2S para ADC");
        return;
    }
    if (i2s_set_clk(i2s_num_adc, 16000, I2S_BITS_PER_SAMPLE_16BIT, I2S_CHANNEL_MONO) != ESP_OK) {
        Serial.println("Error configurando reloj I2S para ADC");
        return;
    }

    // Inicializar I2S para enviar datos al DAC
    if (i2s_driver_install(i2s_num_dac, &i2s_config_dac, 0, NULL) != ESP_OK) {
        Serial.println("Error instalando I2S para DAC");
        return;
    }
    if (i2s_set_pin(i2s_num_dac, &pin_config_dac) != ESP_OK) {
        Serial.println("Error configurando pines I2S para DAC");
        return;
    }
    if (i2s_set_clk(i2s_num_dac, 16000, I2S_BITS_PER_SAMPLE_16BIT, I2S_CHANNEL_MONO) != ESP_OK) {
        Serial.println("Error configurando reloj I2S para DAC");
        return;
    }
}
```
### **- Estructura del Loop**
En el loop es declara les variables necessaries per llegir, processar i enviar la senyal. Després de captar la senyal a través de l'ADC, li aplica el filtre de l'overdrice i l'enivia cap a DAC. Avans de seguir amb el bucle li aplica un delay corresponent a la frequencia de mostreig. 

```cpp
// Leer la señal de audio desde el ADC externo utilizando I2S
    int16_t audioSample;
    size_t bytesRead;
    esp_err_t read_err = i2s_read(i2s_num_adc, &audioSample, sizeof(audioSample), &bytesRead, portMAX_DELAY);
    if (read_err != ESP_OK) {
        Serial.printf("Error leyendo datos I2S: %s\n", esp_err_to_name(read_err));
    } else if (bytesRead != sizeof(audioSample)) {
        Serial.println("Error: No se leyeron suficientes bytes");
    } else {
        // Aplicar el filtro de overdrive a la señal de audio
        int16_t processedSample = applyOverdrive(audioSample);

        // Enviar la señal procesada al DAC utilizando I2S
        size_t bytesWritten;
        esp_err_t write_err = i2s_write(i2s_num_dac, &processedSample, sizeof(processedSample), &bytesWritten, portMAX_DELAY);
        if (write_err != ESP_OK) {
            Serial.printf("Error escribiendo datos I2S: %s\n", esp_err_to_name(write_err));
        } else if (bytesWritten != sizeof(processedSample)) {
            Serial.println("Error: No se escribieron suficientes bytes");
        }
    }

    // Esperar el tiempo correspondiente a la frecuencia de muestreo
    delayMicroseconds(1000000 / 16000);
```
