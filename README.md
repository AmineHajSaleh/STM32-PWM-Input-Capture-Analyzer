# ⚡ STM32 PWM Generator & Input Capture Analyzer

Ce projet implémente un générateur de signal PWM et un analyseur de signal intégré sur un microcontrôleur STM32. Il utilise les Timers matériels avancés pour générer une onde et mesurer précisément sa fréquence ainsi que son rapport cyclique (Duty Cycle) en temps réel via le mode Input Capture et les interruptions.

## 🛠️ Outils et Technologies
* **Cible Matérielle :** Carte de développement STM32
* **IDE :** Keil µVision / ARM-MDK
* **Configuration Matérielle :** STM32CubeMX
* **Bibliothèque :** STM32 HAL (Hardware Abstraction Layer)

## ⚙️ Architecture du Projet

Le projet repose sur l'interaction de deux Timers distincts :
1. **Timer 1 (TIM1) - Génération PWM :** Génère un signal de test dont la fréquence (via `ARR`) et le rapport cyclique (via `CCR1`) sont configurables.
2. **Timer 2 (TIM2) - Input Capture :** Mesure le signal généré de manière asynchrone. 
   * Configuré en **Slave Mode (Reset Mode)** sur la source `TI1FP1`.
   * Le **Channel 1** capture la période complète (déclenchement sur front montant et remise à zéro du compteur).
   * Le **Channel 2** capture la durée de l'impulsion à l'état haut (déclenchement sur front descendant).

## 🔌 Câblage et Routage

Pour que le Timer d'Input Capture puisse mesurer le signal généré par le PWM, une connexion physique (Jumper) est indispensable à l'extérieur de la carte pour relier la sortie à l'entrée.

| Signal | Périphérique | Broche | Connexion |
| :--- | :--- | :--- | :--- |
| **Sortie PWM** | `TIM1_CH1` | Broche de sortie PWM | ➡️ Connecter via Jumper |
| **Entrée IC** | `TIM2_CH1` | `PA0` | ⬅️ Recevoir le signal |

> ⚠️ **Note de débogage (Conflit avec le Bouton User) :**
> Sur certaines cartes de développement STM32 (ex: Discovery), la broche `PA0` est physiquement partagée avec le bouton poussoir "User" (bouton bleu). Si vous utilisez ce projet sur ce type de carte, l'appui sur le bouton générera des signaux qui perturberont la mesure. Veillez à relier la sortie PWM directement sur `PA0` et à ne pas manipuler le bouton pendant l'analyse du signal PWM.

## 💻 Implémentation du Code (Callback)

Le traitement mathématique s'effectue dans le Callback d'interruption du Timer. Les variables sont déclarées avec le mot-clé `volatile` pour éviter l'optimisation par le compilateur et garantir leur observation en temps réel via un débogueur.

```c
/* Variables globales pour l'affichage en mode Debug (Watch) */
volatile uint32_t ICValue = 0;
volatile uint32_t Frequency = 0;
volatile float Duty = 0; 

void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    // Vérification que l'interruption provient de la capture de la période (Channel 1 sur PA0)
    if (htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1) 
    {
        // Lecture de la valeur de la période (Timer Reset Mode)
        ICValue = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1); 
        
        if(ICValue != 0)
        {
            // Lecture du temps à l'état haut (Pulse width sur Channel 2)
            uint32_t pulse_width = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_2);
            
            // Calcul du rapport cyclique (Cast en float obligatoire pour éviter la division entière)
            Duty = ((float)pulse_width * 100.0f) / (float)ICValue;
            
            // Calcul de la fréquence (Horloge système définie à 72MHz / Période)
            Frequency = 72000000 / ICValue;
        }    
    }
}
