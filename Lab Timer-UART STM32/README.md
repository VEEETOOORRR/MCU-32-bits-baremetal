# Gerador de Senoide DDS via PWM e UART (STM32F407VET6)

Projeto referente ao Laboratório de Timer e UART, implementando um gerador de onda senoidal de frequência variável utilizando síntese digital direta (**DDS**), modulação por largura de pulso (**PWM**), filtro passa-baixas **RC** e controle em tempo real via comunicação serial **UART**.

**Vídeo de demonstração:** [Link do YouTube](#)

---

## Visão Geral e Arquitetura

O sistema utiliza o microcontrolador **STM32F407VET6** operando com clock de sistema (`SYSCLK`) a **168 MHz** e barramento de timers APB1 a **84 MHz**.

* **Síntese Digital Direta (DDS):** Uma tabela (*Look-Up Table*) de **1024 amostras** (`LUT_SIZE`) armazena um ciclo completo da senoide mapeado para a resolução do PWM (`0` a `2099`). Um acumulador de fase de 32 bits (`phase_acc`) percorre a tabela a uma taxa fixa de **40 kHz** através da interrupção de *Update* do **TIM3**, atualizando o registrador `TIM3->CCR1` sem bloquear o loop principal.
* **Portadora PWM:** Gerada pelo **TIM3 (Canal 1)** com `Prescaler = 0` e `Period (ARR) = 2099`, resultando em uma portadora fixa de **40 kHz** com resolução superior a 11 bits (2100 níveis).
* **Comunicação Serial (UART):** A **USART1** opera a **115200 bps** (8N1) utilizando recepção não-bloqueante por interrupção (`HAL_UART_Receive_IT`), permitindo alterar a frequência da senoide em tempo real sem interromper a geração do sinal.

---

## Circuito de Filtragem (Passa-Baixas RC)

Para demodular o sinal PWM de 40 kHz e extrair o sinal senoidal analógico ($0\text{ V}$ a $3,3\text{ V}$, com *offset* DC em $1,65\text{ V}$), utilizou-se um filtro passa-baixas RC de 1ª ordem na saída do **TIM3_CH1**:

* **Resistor ($R$):** $1\text{ k}\Omega$
* **Capacitor ($C$):** $100\text{ nF}$ (cerâmico/poliéster)

A frequência de corte ($f_c$) resultante é:

$$f_c = \frac{1}{2 \pi R C} = \frac{1}{2 \pi \times 1000 \times 100 \times 10^{-9}} \approx \mathbf{1591,5\text{ Hz}}$$

---

## Faixa de Operação e Limites de Frequência

* **Intervalo configurado no software:** **1 Hz a 5000 Hz** (com frequência inicial padrão de **60 Hz**).
* **Faixa ideal recomendada para este filtro RC:** **10 Hz a ~500 Hz** (podendo operar com atenuação progressiva até o limite de corte em ~1590 Hz).
  * *Abaixo de ~1590 Hz:* A amplitude pico a pico mantém-se próxima de $3,3\text{ V}_{\text{pp}}$ com boa atenuação do *ripple* de 40 kHz.
  * *Acima de ~1590 Hz:* O filtro passa a atenuar a amplitude da fundamental (queda de $-20\text{ dB/década}$) e a quantidade de amostras por ciclo diminui.

Para trabalhar em faixas de frequência diferentes sem perda de amplitude, basta recalcular os valores de $R$ e $C$ e ajustar os limites de validação no `main.c`.

---

## Como Utilizar

1. Conecte um conversor USB-Serial aos pinos da **USART1** (TX/RX) da placa STM32F407VET6 e ligue o filtro RC na saída do **TIM3_CH1** (`PA6`).
2. Abra um terminal serial (ex: `tio`, PuTTY ou CuteCom) configurado para **115200 bps**:
   ```bash
   tio -b 115200 --local-echo /dev/ttyUSB0