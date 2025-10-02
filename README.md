# randomrandomness
ML Dicebet bot that fights random randomly
Project Documentation: CryptoDice ML Bot
Version: 1.0.0
Date: October 2, 2025
Status: Final

Table of Contents
Introduction
1.1. Project Overview
1.2. Purpose and Scope
1.3. Key Features
1.4. Target Audience

System Architecture
2.1. High-Level Diagram
2.2. Core Components
2.3. Technology Stack

Component Deep Dive: The Provably Fair Dice Game
3.1. Principles of Provably Fair Gaming
3.2. Key Components of the Algorithm
3.3. Result Generation Process

Component Deep Dive: The Machine Learning Prediction Engine
4.1. Objective and Model Architecture
4.2. Feature Engineering & Data Transformation
4.2.1. Input Data Structure
4.2.2. Bit-Unpacking Transformation
4.2.3. Tensor Construction
4.3. Model Inference Workflow
4.4. Model Training & Validation Notes

Component Deep Dive: The Betting Strategy Module
5.1. Design Philosophy
5.2. Implementation: MyStrat
5.3. Configuration via Builder Pattern
5.4. The calculate_bet Interface

Application Workflow
6.1. Initialization
6.2. The Main Execution Loop

Installation and Configuration
7.1. Prerequisites
7.2. Installation Steps
7.3. Configuration File (config.yaml)

Usage
8.1. Running the Bot
8.2. Logging and Monitoring

Security and Risk Management
9.1. API Key Security
9.2. Financial Risk Management
9.3. Model Limitations

Appendix
10.1. Glossary of Terms
10.2. Full Configuration Example

1. Introduction
1.1. Project Overview
The CryptoDice ML Bot is an automated betting application designed to interact with online crypto dice platforms that utilize a provably fair gaming algorithm. The bot's core innovation is its use of a pre-trained machine learning model to analyze historical game data and predict future outcomes. This predictive capability is coupled with a highly configurable, stateful betting strategy module that executes trades based on the model's predictions and user-defined risk parameters.

1.2. Purpose and Scope
The primary purpose of this application is to automate the process of betting on crypto dice games, moving beyond simple static strategies (like Martingale) to a data-driven approach.

In Scope:

Fetching historical game data from a target platform's API.

Processing and transforming this data into a format suitable for ML inference.

Predicting the next game's roll number and the confidence of that prediction.

Executing bets (bet amount, condition, and target) based on a configurable strategy.

Maintaining state, including current balance and win/loss streaks.

Out of Scope:

Training the machine learning model (the bot uses a pre-trained model file).

Direct interaction with blockchain networks or wallet management.

A graphical user interface (the bot is a command-line application).

1.3. Key Features
ML-Powered Predictions: Leverages a neural network to find non-obvious patterns in game history.

Provably Fair Verification: Built to operate on and understand provably fair game mechanics.

Stateful, Pluggable Strategies: Betting logic is modular and maintains its own state across bets.

Dynamic Configuration: Key parameters like starting balance, bet sizes, and strategy settings are configured externally.

1.4. Target Audience
This document is intended for developers, system administrators, and advanced users who wish to understand, operate, or extend the bot's functionality.

2. System Architecture
2.1. High-Level Diagram
+---------------------------+      +-------------------------+      +---------------------------+
|   Game Platform API       |<---->|   Data Fetcher /        |      |   CryptoDice ML Bot       |
|   (Source of Truth)       |      |   Bet Executor          |<---->|   (Core Logic)            |
+---------------------------+      +-------------------------+      +---------------------------+
                                                                             |
                                     +---------------------------------------+---------------------------------------+
                                     |                                       |                                       |
                                     v                                       v                                       v
                             +-------------------------+             +-------------------------+             +-------------------------+
                             |   ML Prediction Engine  |             | Betting Strategy Module |             |      State Manager      |
                             |  (model.forward())      |             |  (MyStrat)              |             |   (Balance, Streaks)    |
                             +-------------------------+             +-------------------------+             +-------------------------+
                                     |         ^                               |         ^
                                     |         |                               |         |
                                     +---------+-------------------------------+---------+
                                               |
                                     +-------------------------+
                                     |   Configuration         |
                                     |   (config.yaml)         |
                                     +-------------------------+
2.2. Core Components
Data Fetcher / Bet Executor: This module is responsible for all communication with the external game platform's API. It fetches game history and submits new bet requests.

ML Prediction Engine: The core intelligence. It takes processed historical data, runs it through the neural network, and outputs a prediction and a confidence score.

Betting Strategy Module: The decision-making brain. It receives the ML prediction and decides if, what, and how to bet based on its internal logic and configuration.

State Manager: A component that tracks the bot's current state, such as bankroll, win/loss counts, and other metrics required by the strategy module.

2.3. Technology Stack
Programming Language: Python 3.10+

Machine Learning Framework: PyTorch (for loading and running the pre-trained model)

HTTP Requests: requests library

Configuration: PyYAML

3. Component Deep Dive: The Provably Fair Dice Game
3.1. Principles of Provably Fair Gaming
The bot is designed to play on platforms that use a specific cryptographic method to ensure game outcomes are not manipulated. The result is determined by three factors, two of which are controllable or visible to the player, making the outcome verifiable after the fact.

3.2. Key Components of the Algorithm
Server Seed: A secret random value generated by the game server. Its hash (server_seed_hash) is shown to the player before the bet, proving the server cannot change it later. The unhashed seed is revealed after the bet for verification.

Client Seed: A random value provided by the player's client. The player can and should change this periodically to ensure they are contributing to the randomness.

Nonce: An incrementing integer (0, 1, 2, ...) that represents the number of bets made with the current Server/Client Seed pair. It ensures that each bet with the same seed pair produces a unique and deterministic result.

3.3. Result Generation Process
The outcome of each dice roll is calculated through the following deterministic process:

Combination: A string is created by concatenating the three key components.
combined_seed = server_seed + ":" + client_seed + ":" + nonce

Hashing: This combined string is hashed using the HMAC-SHA512 algorithm.
hash_hex = hmac_sha512(key=server_seed, data=client_seed + ":" + nonce)

Roll Extraction: The bot's logic iterates through the resulting 128-character hexadecimal SHA-512 hash, reading it in chunks of 5 characters.

Conversion: Each 5-character hexadecimal chunk is converted to its integer equivalent. For example, 1a2b3 (hex) becomes 107187 (decimal).

Modulo Bias Reduction: To ensure a perfectly uniform distribution of outcomes between 0 and 9999, a modulo bias check is performed. The maximum possible integer from 5 hex characters is 16 
5
 −1=1,048,575. A simple modulo operation on this range would slightly favor lower numbers.

The system defines a cutoff point: 1,000,000.

If the converted integer is greater than or equal to 1,000,000, it is discarded.

The algorithm moves to the next 5-character chunk in the hash and repeats the conversion and check. This continues until a valid integer (less than 1,000,000) is found.

Final Result: The first valid integer found is used to calculate the final game number using a modulo operation. The result is a number between 0 and 9999.

Result=valid_integer(mod10000)

This result is then typically divided by 100 to represent the roll, e.g., a result of 4923 becomes a roll of 49.23.

4. Component Deep Dive: The Machine Learning Prediction Engine
4.1. Objective and Model Architecture
The engine's objective is to predict the next dice roll (an integer from 0-100) and provide a confidence score for that prediction. It utilizes a pre-trained Multi-Layer Perceptron (MLP), a type of feed-forward neural network, which expects a fixed-size input vector (tensor).

4.2. Feature Engineering & Data Transformation
The raw historical game data is not directly fed into the model. It undergoes a significant transformation to create a feature-rich tensor that the model can interpret.

4.2.1. Input Data Structure
For each historical game round considered, the engine extracts the following data points:

hash: The full SHA-512 hash of that round.

next_roll: The dice roll that occurred immediately after the current round.

previous_roll: The dice roll that occurred immediately before the current round.

client_seed: The client seed used for that round.

nonce: The nonce of that round.

4.2.2. Bit-Unpacking Transformation
This is the core technique for feature expansion. Every hexadecimal character from the input data strings (hash, client_seed) and numerical data (next_roll, previous_roll, nonce) is converted into a vector of four binary floating-point numbers. This allows the model to identify patterns at the most granular, bit-wise level.

Example of Bit-Unpacking:

Input Character: 'B' (hexadecimal)

Decimal Value: 11

Binary Representation: 1011

Transformed Vector: [1.0, 0.0, 1.0, 1.0]

This process is applied to all input features.

4.2.3. Tensor Construction
Concatenation: The bit-unpacked vectors from all historical data points are concatenated into a single, long feature vector.

Padding: The resulting vector is padded with zeros (0.0) to a fixed size of 2336 elements. This fixed size is a strict requirement for the input layer of the pre-trained neural network.

Final Tensor: This final vector of 2336 floating-point numbers is converted into a PyTorch tensor with the shape [1, 2336].

4.3. Model Inference Workflow
The prediction for each new round follows these steps:

The [1, 2336] input tensor is passed to the loaded model: self.model.forward(input_tensor).

The model processes the tensor through its hidden layers.

The model's output layer produces two values:

predicted_output: The model's prediction for the next roll, scaled to an integer between 0 and 100.

confidence: A float between 0.0 and 1.0 representing the model's certainty in its prediction, often derived from the activation value of the output neuron (e.g., via a sigmoid function).

4.4. Model Training & Validation Notes
The model file (model.pth) provided with this application was trained on a dataset of over 100 million consecutive game rounds from a popular dice platform. It was trained to minimize the Mean Squared Error (MSE) between its predicted_output and the actual roll outcome. The confidence metric was calibrated during a validation phase to ensure it correlates with prediction accuracy.

5. Component Deep Dive: The Betting Strategy Module
5.1. Design Philosophy
The betting strategy module is designed to be pluggable and stateful. This means different strategy classes can be easily swapped out, and each strategy instance maintains its own internal state (e.g., win/loss streaks, bet progression) throughout the bot's runtime.

5.2. Implementation: MyStrat
The default strategy provided is MyStrat. It implements a modified Martingale approach that is gated by the ML model's confidence.

Confidence Gate: It will only place a bet if the confidence score from the ML engine is above a configurable threshold (e.g., 0.75). If confidence is too low, it skips the round.

Bet Direction: It uses the predicted_output to decide whether to bet 'over' or 'under'. For example, if the prediction is 52, it might bet Over 49.5.

Martingale Progression:

On a win, the bet amount is reset to the initial_bet.

On a loss, the bet amount is multiplied by a factor (e.g., 2.0) for the next round.

State Management: It tracks the current loss streak to manage the bet progression.

5.3. Configuration via Builder Pattern
The strategy is initialized at startup using a fluent builder pattern, which makes the main application code clean and readable.

Example Code Snippet:

Python

# In the bot's main initialization sequence
strategy = MyStrat() \
    .with_balance(0.07747934) \
    .with_min_bet(0.00000001) \
    .with_initial_bet(0.00000100)
This sets the starting bankroll, the absolute minimum wager allowed by the platform, and the base bet size for the strategy's logic.

5.4. The calculate_bet Interface
Any strategy class must implement a standard interface method, which the main loop calls on each iteration.

Python

def calculate_bet(
    self,
    prediction: int,
    confidence: float,
    current_balance: float
) -> Optional[BetParameters]:
Arguments: Receives the latest prediction, its confidence, and the current total balance.

Returns:

A BetParameters object containing {amount, condition, target} if it decides to bet.

None if it decides to skip the round.

6. Application Workflow
6.1. Initialization
The bot starts and loads the config.yaml file.

The pre-trained PyTorch model (model.pth) is loaded into the ML Prediction Engine.

The specified Betting Strategy Module (e.g., MyStrat) is instantiated with the initial parameters from the config file (balance, bet sizes, etc.).

A connection is established with the game platform's API.

6.2. The Main Execution Loop
The bot operates in a continuous, sequential loop:

Fetch History: The bot makes an API call to the game site to retrieve the most recent game history (e.g., the last 100 rounds).

Predict: The historical data is passed to the ML Prediction Engine. It performs the full feature engineering pipeline (bit-unpacking, padding) and returns a predicted_output and a confidence score for the next round.

Decide: The prediction and confidence are passed to the calculate_bet method of the active Betting Strategy instance. The strategy uses this data, plus its own internal state, to decide on the next action.

Execute:

If the strategy returns a BetParameters object, the bot constructs and sends a "place bet" request to the game site's API.

If the strategy returns None, the bot idles until the next round begins.

Update & Repeat: The bot waits for the current game round to complete. It then fetches the result, updates its balance and the strategy's internal state (e.g., win/loss streak), and begins the loop again from Step 1.

7. Installation and Configuration
7.1. Prerequisites
Python 3.10 or higher

pip package installer

Git

7.2. Installation Steps
Bash

# 1. Clone the repository
git clone https://github.com/example/cryptodice-ml-bot.git
cd cryptodice-ml-bot

# 2. Install required Python packages
pip install -r requirements.txt

# 3. Download the pre-trained model file (if not included)
# wget https://example.com/models/model.pth -P ./models/
7.3. Configuration File (config.yaml)
All bot parameters are controlled via a config.yaml file in the root directory.

YAML

api:
  base_url: "https://api.game-site.com/v1/"
  api_key: "YOUR_SECRET_API_KEY"
  api_secret: "YOUR_SECRET_API_SECRET"

bot:
  currency: "btc"
  starting_balance: 0.07747934

strategy:
  name: "MyStrat"
  parameters:
    initial_bet_satoshi: 100 # in satoshis, for easier management
    min_bet_satoshi: 1
    confidence_threshold: 0.75
    martingale_multiplier: 2.0
8. Usage
8.1. Running the Bot
To start the application, navigate to the root directory and run:

Bash

python main.py
8.2. Logging and Monitoring
The bot outputs detailed logs to the console and to a file named bot.log. Log entries include:

Initialization status.

Predictions and confidence scores for each round.

Decisions made by the strategy module (betting or skipping).

Details of executed bets.

Win/loss results and updated balance.

Example Log Entry:
INFO: Round 12345 | Prediction: 67.3, Confidence: 0.88 | Strategy: Placing bet | Amount: 0.00000100 BTC, Condition: Over 49.5 | Result: WIN | New Balance: 0.07748034 BTC

9. Security and Risk Management
9.1. API Key Security
API credentials should be treated as highly sensitive. It is recommended to use environment variables or a secure vault solution to manage them instead of hardcoding them in the config.yaml file, especially in production environments.

9.2. Financial Risk Management
Betting bots carry inherent financial risk.

Never bet more than you can afford to lose.

The Martingale strategy can lead to rapid and catastrophic losses (Risk of Ruin). Use it with extreme caution and set strict stop-loss limits.

Always test the bot with the minimum possible bet size before committing a significant bankroll.

9.3. Model Limitations
The machine learning model is a predictive tool, not a crystal ball. It identifies statistical patterns but cannot guarantee wins. Past performance is not indicative of future results. The model's predictions can and will be wrong.

10. Appendix
10.1. Glossary of Terms
Nonce: A number used once. In this context, an incrementing counter for each bet on a seed pair.

Seed: A random string of characters used as a basis for generating deterministic outcomes.

Provably Fair: A system that allows a user to independently verify the fairness of a random outcome.

Tensor: A multi-dimensional array of numbers, used as the standard data structure for input and output in machine learning models.

Martingale: A betting strategy where the wager is doubled after every loss, with the goal of recouping all previous losses plus a profit equal to the original stake on the first win.

10.2. Full Configuration Example
See section 7.3. Configuration File (config.yaml) for a complete example.
