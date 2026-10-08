#include <stdint.h>

#define REG32(address) (*(volatile uint32_t *)(address))

#define RCC_AHB1ENR   REG32(0x40023830U)

#define GPIOA_MODER   REG32(0x40020000U)
#define GPIOA_PUPDR   REG32(0x4002000CU)
#define GPIOA_IDR     REG32(0x40020010U)

#define GPIOD_MODER   REG32(0x40020C00U)
#define GPIOD_OTYPER  REG32(0x40020C04U)
#define GPIOD_OSPEEDR REG32(0x40020C08U)
#define GPIOD_PUPDR   REG32(0x40020C0CU)
#define GPIOD_BSRR    REG32(0x40020C18U)

#define SYSTICK_CTRL  REG32(0xE000E010U)
#define SYSTICK_LOAD  REG32(0xE000E014U)
#define SYSTICK_VAL   REG32(0xE000E018U)

#define GREEN_LED (1U << 12)
#define BLUE_LED  (1U << 15)

#define DEBOUNCE_MS    20U
#define DOUBLE_CLICK_MS 350U
#define LONG_PRESS_MS  1000U

typedef enum
{
    MODE_OFF,
    MODE_SLOW,
    MODE_FAST,
    MODE_ON
} LedMode;

static LedMode mode = MODE_OFF;
static uint32_t green_on = 0U;
static uint32_t pattern_ms = 0U;

static void green_write(uint32_t on)
{
    green_on = on;

    if (on != 0U)
        GPIOD_BSRR = GREEN_LED;
    else
        GPIOD_BSRR = GREEN_LED << 16;
}

static void set_mode(LedMode next)
{
    mode = next;
    pattern_ms = 0U;

    /* Start each active pattern with the green LED on */
    green_write(next != MODE_OFF);
}

static void pattern_tick(void)
{
    if ((mode == MODE_SLOW) || (mode == MODE_FAST))
    {
        uint32_t interval = (mode == MODE_SLOW) ? 500U : 100U;

        pattern_ms++;

        if (pattern_ms >= interval)
        {
            pattern_ms = 0U;
            green_write(green_on ^ 1U);
        }
    }
}

int main(void)
{
    RCC_AHB1ENR |= (1U << 0) | (1U << 3);
    (void)RCC_AHB1ENR;

    GPIOD_BSRR = (GREEN_LED | BLUE_LED) << 16;

    /* Green PD12 and blue PD15: push-pull outputs */
    GPIOD_MODER &= ~((3U << 24) | (3U << 30));
    GPIOD_MODER |=   (1U << 24) | (1U << 30);
    GPIOD_OTYPER &= ~(GREEN_LED | BLUE_LED);
    GPIOD_OSPEEDR &= ~((3U << 24) | (3U << 30));
    GPIOD_PUPDR &= ~((3U << 24) | (3U << 30));

    /* USER button PA0: input with pull-down */
    GPIOA_MODER &= ~3U;
    GPIOA_PUPDR = (GPIOA_PUPDR & ~3U) | 2U;

    /* 1 ms polling tick at the unchanged 16 MHz reset clock */
    SYSTICK_CTRL = 0U;
    SYSTICK_LOAD = 16000U - 1U;
    SYSTICK_VAL = 0U;
    SYSTICK_CTRL = (1U << 2) | (1U << 0);

    uint32_t now = 0U;
    uint32_t raw_previous = 0U;
    uint32_t stable_button = 0U;
    uint32_t raw_changed_at = 0U;
    uint32_t pressed_at = 0U;
    uint32_t released_at = 0U;
    uint32_t long_handled = 0U;
    uint32_t click_pending = 0U;
    uint32_t second_press = 0U;

    while (1)
    {
        if ((SYSTICK_CTRL & (1U << 16)) == 0U)
            continue;

        now++;

        uint32_t raw = GPIOA_IDR & 1U;

        /* Restart the debounce timer whenever the input changes */
        if (raw != raw_previous)
        {
            raw_previous = raw;
            raw_changed_at = now;
        }

        /* Accept a change only after 20 ms of stable input */
        if ((raw != stable_button) &&
            ((uint32_t)(now - raw_changed_at) >= DEBOUNCE_MS))
        {
            stable_button = raw;

            if (stable_button != 0U)
            {
                /* Debounced press */
                GPIOD_BSRR = BLUE_LED;
                pressed_at = now;
                long_handled = 0U;
                second_press = 0U;

                if (click_pending != 0U)
                {
                    if ((uint32_t)(now - released_at) <=
                        DOUBLE_CLICK_MS)
                    {
                        second_press = 1U;
                    }
                    else
                    {
                        set_mode(MODE_SLOW);
                    }

                    click_pending = 0U;
                }
            }
            else
            {
                /* Debounced release */
                GPIOD_BSRR = BLUE_LED << 16;

                if (long_handled == 0U)
                {
                    if ((uint32_t)(now - pressed_at) >=
                        LONG_PRESS_MS)
                    {
                        set_mode(MODE_ON);
                    }
                    else if (second_press != 0U)
                    {
                        set_mode(MODE_FAST);
                    }
                    else
                    {
                        click_pending = 1U;
                        released_at = now;
                    }
                }

                second_press = 0U;
            }
        }

        /* Recognize a long press while the button is still held */
        if ((stable_button != 0U) &&
            (long_handled == 0U) &&
            ((uint32_t)(now - pressed_at) >= LONG_PRESS_MS))
        {
            long_handled = 1U;
            click_pending = 0U;
            second_press = 0U;
            set_mode(MODE_ON);
        }

        /* No second press arrived: accept the single click */
        if ((click_pending != 0U) &&
            (stable_button == 0U) &&
            ((uint32_t)(now - released_at) > DOUBLE_CLICK_MS))
        {
            click_pending = 0U;
            set_mode(MODE_SLOW);
        }

        /* Patterns continue while we monitor the button */
        pattern_tick();
    }
}
