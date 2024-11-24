#include <stdio.h>
#include <string.h>
#include <stdbool.h>
#include <stdlib.h>
#include <time.h>

#define OTP_BLOCK_TIME 86400   // 24 hours in seconds
#define CVV_BLOCK_TIME 86400   // 24 hours in seconds

// Structure for Credit Card details
struct CreditCard {
    char cardNumber[17];  // Credit card number (16 digits)
    char cardholderName[50];
    char expiryDate[6];   // Format MM/YY
    int cvv;              // CVV for online transactions
    float creditLimit;     // Custom allowed credit limit
    float moneySpent;      // Total money spent so far
    float currentBalance;  // Current available balance after spending (credit limit - money spent)
    int pin;              // PIN for accessing card
    bool isBlocked;
    int failedOTPAttempts;
    int failedCVVAttempts;
    time_t blockStartTime;
    float rewardPoints;    // Reward points for transactions
};

// Function prototypes
void registerCard(struct CreditCard *cc);
bool validatePin(const struct CreditCard *cc);
void displayCardDetails(const struct CreditCard *cc);
void makeTransaction(struct CreditCard *cc, float amount, bool isOnline, const char *category);
void makePayment(struct CreditCard *cc, float amount);
void checkBalance(const struct CreditCard *cc);
void sendEmail(const struct CreditCard *cc, const char *message);
void sendSMS(const struct CreditCard *cc, const char *message);
int generateOTP();
bool validateCVV(struct CreditCard *cc, int inputCVV);
void blockCard(struct CreditCard *cc);
bool isCardBlocked(struct CreditCard *cc);
void exitIfBlocked(struct CreditCard *cc);

// New feature function prototypes
void sendSpendCategoryAlert(const char *category, float amount);
void adjustCreditLimit(struct CreditCard *cc);
void trackRewardPoints(struct CreditCard *cc, float amount);
void showRewardsAndInsights(const struct CreditCard *cc);

int main() {
    struct CreditCard cc;
    int choice;
    float amount;
    char category[50];
    int attempts = 0;

    // Register a new card
    registerCard(&cc);

    // PIN validation loop (3 attempts)
    while (!validatePin(&cc)) {
        attempts++;
        if (attempts == 3) {
            printf("Too many wrong PIN attempts! Access blocked.\n");
            return 0; // Exit program
        }
    }

    // Menu loop
    do {
        exitIfBlocked(&cc);  // Check if card is blocked before every operation

        printf("\n--- Credit Card Management System ---\n");
        printf("1. Display Card Details\n");
        printf("2. Make an Online Transaction\n");
        printf("3. Make a Payment (EMI)\n");
        printf("4. Check Balance\n");
        printf("5. Send Balance Sheet via Email\n");
        printf("6. Show Rewards & Insights\n");
        printf("7. Exit\n");
        printf("Enter your choice: ");
        scanf("%d", &choice);

        switch (choice) {
            case 1:
                displayCardDetails(&cc);
                break;
            case 2:
                printf("Enter transaction amount: ");
                scanf("%f", &amount);
                printf("Enter transaction category (e.g., Shopping, Grocery, Entertainment): ");
                scanf("%s", category);
                makeTransaction(&cc, amount, true, category);  // Online transaction with category
                break;
            case 3:
                printf("Enter payment amount (EMI): ");
                scanf("%f", &amount);
                makePayment(&cc, amount);
                break;
            case 4:
                checkBalance(&cc);
                break;
            case 5:
                sendEmail(&cc, "Your updated balance sheet.");
                break;
            case 6:
                showRewardsAndInsights(&cc);
                break;
            case 7:
                printf("Exiting...\n");
                break;
            default:
                printf("Invalid choice! Please try again.\n");
        }
    } while (choice != 7);

    return 0;
}

// Function to register a new credit card
void registerCard(struct CreditCard *cc) {
    printf("Enter credit card number (16 digits): ");
    scanf("%s", cc->cardNumber);

    printf("Enter cardholder name: ");
    scanf(" %[^\n]s", cc->cardholderName);

    printf("Enter expiry date (MM/YY): ");
    scanf("%s", cc->expiryDate);

    printf("Enter CVV (3 digits): ");
    scanf("%d", &cc->cvv);

    printf("Enter credit limit: ");
    scanf("%f", &cc->creditLimit);

    printf("Set a 4-digit PIN: ");
    scanf("%d", &cc->pin);

    cc->currentBalance = cc->creditLimit;     // Initialize current balance to credit limit
    cc->moneySpent = 0;                       // Initialize money spent to 0
    cc->isBlocked = false;                    // Card initially unblocked
    cc->failedOTPAttempts = 0;                // Initialize failed OTP attempts to zero
    cc->failedCVVAttempts = 0;                // Initialize failed CVV attempts to zero
    cc->rewardPoints = 0;                     // Initialize reward points to zero

    printf("Card registered successfully with a credit limit of %.2f!\n", cc->creditLimit);
}

// Function to validate the card's PIN
bool validatePin(const struct CreditCard *cc) {
    int inputPin;
    printf("Enter your 4-digit PIN: ");
    scanf("%d", &inputPin);

    if (inputPin == cc->pin) {
        printf("PIN validated successfully!\n");
        return true;
    } else {
        printf("Incorrect PIN! Please try again.\n");
        return false;
    }
}

// Function to display card details
void displayCardDetails(const struct CreditCard *cc) {
    printf("\n--- Credit Card Details ---\n");
    printf("Card Number: %s\n", cc->cardNumber);
    printf("Cardholder Name: %s\n", cc->cardholderName);
    printf("Expiry Date: %s\n", cc->expiryDate);
    printf("Credit Limit: %.2f\n", cc->creditLimit);
    printf("Current Balance (Remaining Credit): %.2f\n", cc->creditLimit - cc->moneySpent);
    printf("Reward Points: %.2f\n", cc->rewardPoints);
}

// Function to make an online transaction
void makeTransaction(struct CreditCard *cc, float amount, bool isOnline, const char *category) {
    exitIfBlocked(cc);  // Check if card is blocked before making a transaction

    if ((cc->creditLimit - cc->moneySpent) - amount < 0) {
        printf("Transaction declined! Exceeds available balance.\n");
        return;
    }

    if (isOnline) {
        int otp, cvv;
        printf("Enter CVV: ");
        scanf("%d", &cvv);

        // Validate CVV first
        if (!validateCVV(cc, cvv)) {
            printf("Transaction declined due to incorrect CVV.\n");
            return;  // Exit if CVV is incorrect
        }

        printf("CVV validated successfully. Now validating OTP...\n");

        // Generate and validate OTP
        int generatedOTP = generateOTP();
        printf("Generated OTP: %d (Simulated, for demonstration purposes)\n", generatedOTP);

        // Allow 3 attempts for OTP within the same transaction
        for (int attempt = 0; attempt < 3; attempt++) {
            printf("Enter OTP: ");
            scanf("%d", &otp);

            if (otp == generatedOTP) {
                printf("OTP validated successfully. Proceeding with transaction...\n");
                break;  // Exit loop if OTP is correct
            } else {
                printf("Incorrect OTP! You have %d attempts left.\n", 2 - attempt);
                if (attempt == 2) {
                    printf("Transaction declined due to incorrect OTP.\n");
                    cc->failedOTPAttempts++;
                    if (cc->failedOTPAttempts == 3) {
                        printf("Too many incorrect OTP attempts. Card blocked for 24 hours!\n");
                        blockCard(cc);
                        exit(0);  // Exit the program after the card is blocked
                    }
                    return;  // Exit if OTP is incorrect after 3 attempts
                }
            }
        }
    }

    // Transaction proceeds if both CVV and OTP are correct
    cc->moneySpent += amount;
    cc->currentBalance = cc->creditLimit - cc->moneySpent;
    printf("Transaction of %.2f in %s category successful. New remaining balance: %.2f\n", amount, category, cc->currentBalance);

    // Track reward points and insights for the transaction
    trackRewardPoints(cc, amount);

    // Send Spend Category Alerts
    sendSpendCategoryAlert(category, amount);

    // Adjust credit limit dynamically based on behavior (simulated)
    adjustCreditLimit(cc);

    // Send SMS with the new balance
    char message[100];
    sprintf(message, "Transaction successful! Your remaining balance is %.2f.", cc->currentBalance);
    sendSMS(cc, message);  // Send SMS with the updated balance

    // Send an email with the updated balance sheet
    sendEmail(cc, "Your updated balance sheet after transaction.");
}

// Function to make a payment (EMI)
void makePayment(struct CreditCard *cc, float amount) {
    exitIfBlocked(cc);  // Check if card is blocked before making a payment

    if (amount > cc->moneySpent) {
        printf("Payment exceeds the amount spent. Transaction declined!\n");
        return;
    }

    cc->moneySpent -= amount;  // Deduct the payment from money spent
    cc->currentBalance = cc->creditLimit - cc->moneySpent;  // Update current balance
    printf("Payment of %.2f successful! New remaining balance: %.2f\n", amount, cc->currentBalance);
}

// Function to check current balance
void checkBalance(const struct CreditCard *cc) {
    printf("Current available balance: %.2f\n", cc->creditLimit - cc->moneySpent);
}

// Function to send balance sheet via email
void sendEmail(const struct CreditCard *cc, const char *message) {
    printf("Sending email to %s with message: %s\n", cc->cardholderName, message);
}

// Function to send SMS notification
void sendSMS(const struct CreditCard *cc, const char *message) {
    printf("Sending SMS to cardholder with message: %s\n", message);
}

// Function to generate a random OTP
int generateOTP() {
    srand(time(0));
    return 100000 + rand() % 900000;  // Generate a 6-digit OTP
}

// Function to validate CVV
bool validateCVV(struct CreditCard *cc, int inputCVV) {
    if (inputCVV == cc->cvv) {
        return true;
    } else {
        cc->failedCVVAttempts++;
        if (cc->failedCVVAttempts == 3) {
            printf("Too many incorrect CVV attempts. Card blocked for 24 hours!\n");
            blockCard(cc);
            exit(0);  // Exit program after the card is blocked
        }
        return false;
    }
}

// Function to block the card
void blockCard(struct CreditCard *cc) {
    cc->isBlocked = true;
    cc->blockStartTime = time(NULL);  // Record the block time
}

// Function to check if the card is blocked
bool isCardBlocked(struct CreditCard *cc) {
    if (cc->isBlocked) {
        time_t currentTime = time(NULL);
        if (difftime(currentTime, cc->blockStartTime) >= OTP_BLOCK_TIME) {
            cc->isBlocked = false;  // Unblock the card after 24 hours
            cc->failedOTPAttempts = 0; // Reset failed attempts
            cc->failedCVVAttempts = 0; // Reset failed attempts
            printf("Card is now unblocked.\n");
        }
    }
    return cc->isBlocked;
}
// Function to check if the card is blocked before each transaction
void exitIfBlocked(struct CreditCard *cc) {
    if (isCardBlocked(cc)) {
        printf("Card is currently blocked. Please try again later.\n");
        exit(0);  // Exit the program if the card is blocked
    }
}
// New feature: Send spend category alerts
void sendSpendCategoryAlert(const char *category, float amount) {
    printf("Alert: You spent %.2f in the %s category.\n", amount, category);
}
// New feature: Adjust credit limit dynamically (simulated)
void adjustCreditLimit(struct CreditCard *cc) {
    if (cc->moneySpent > cc->creditLimit * 0.8) {
        cc->creditLimit *= 0.9; // Decrease limit if over 80% spent
        printf("Credit limit adjusted down to %.2f due to high spending.\n", cc->creditLimit);
    }
}

// New feature: Track reward points
void trackRewardPoints(struct CreditCard *cc, float amount) {
    float pointsEarned = amount * 0.01; // Earn 1 point for every 100 spent
    cc->rewardPoints += pointsEarned;
    printf("You earned %.2f reward points!\n", pointsEarned);
}
// New feature: Show rewards and insights
void showRewardsAndInsights(const struct CreditCard *cc) {
    printf("Total reward points: %.2f\n", cc->rewardPoints);
    printf("Total spent: %.2f\n", cc->moneySpent);
}

