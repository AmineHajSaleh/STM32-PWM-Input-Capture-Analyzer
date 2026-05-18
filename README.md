# STM32 PWM Generator & Input Capture Analyzer

Ce projet implémente un générateur de signal PWM et un analyseur de signal intégré sur un microcontrôleur STM32. Il utilise les Timers matériels pour générer une onde et mesurer précisément sa fréquence ainsi que son rapport cyclique (Duty Cycle) en temps réel via des interruptions.

## 🛠️ Outils et Technologies
* **Cible Matérielle :** Carte de développement STM32 (ex: STM32F4 / Discovery / Nucleo)
* **IDE :** Keil µVision / ARM-MDK
* **Générateur de code :** STM32CubeMX
* **Bibliothèque :** STM32 HAL (Hardware Abstraction Layer)

## ⚙️ Architecture du Projet

Le projet repose sur la configuration de deux Timers distincts :
1. **Timer 1 (TIM1) - Mode PWM Generation :** Génère le signal de test avec une fréquence et un rapport cyclique configurables via le registre `CCR1`.
2. **Timer 2 (TIM2) - Mode Input Capture :** Mesure le signal généré. 
   * Configuré en **Slave Mode (Reset Mode)** sur la source `TI1FP1`.
   * Le **Channel 1** capture la période complète (déclenchement sur front montant).
   * Le **Channel 2** capture le temps à l'état haut (déclenchement sur front descendant).

## 🔌 Câblage Requis (Important)

Pour que l'Input Capture puisse mesurer le signal généré par le PWM, une connexion physique (Jumper) est indispensable à l'extérieur de la carte.

| Signal | Périphérique STM32 | Broche (Exemple) | Connexion |
| :--- | :--- | :--- | :--- |
| **Sortie PWM** | TIM1_CH1 | `PA8` (ou `PE9`) | ➡️ Connecter à l'entrée IC |
| **Entrée IC** | TIM2_CH2 | `PA1` | ⬅️ Recevoir depuis le PWM |

> ⚠️ **Note sur le multiplexage des broches (Le piège du Bouton User) :**
> Par défaut, `TIM2_CH1` est souvent mappé sur la broche `PA0`. Sur de nombreuses cartes STM32, `PA0` est physiquement reliée au bouton poussoir "User" (Bouton Bleu). Si vous utilisez `PA0` pour l'Input Capture, l'appui sur le bouton générera des signaux parasites qui fausseront les mesures. Il est recommandé de mapper l'Input Capture sur un autre canal (ex: `TIM2_CH2` sur `PA1`) pour isoler le signal d'entrée.

## 💻 Code Principal (Interruption)

Le calcul s'effectue de manière asynchrone dans le Callback d'interruption du Timer. Les variables sont déclarées `volatile` pour garantir leur mise à jour en mode Debug.

```c
/* Variables globales pour l'affichage en mode Debug */
volatile uint32_t ICValue = 0;
volatile uint32_t Frequency = 0;
volatile float Duty = 0; 

void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1) 
    {
        // Lecture de la valeur de la période (Timer Reset Mode)
        ICValue = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1); 
        
        if(ICValue != 0)
        {
            // Lecture du temps à l'état haut
            uint32_t pulse_width = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_2);
            
            // Calcul du rapport cyclique (Cast en float obligatoire)
            Duty = ((float)pulse_width * 100.0f) / (float)ICValue;
            
            // Calcul de la fréquence (Horloge à 72MHz / Période)
            Frequency = 72000000 / ICValue;
        }    
    }
}
