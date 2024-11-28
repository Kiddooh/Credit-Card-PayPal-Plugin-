<?php
require 'vendor/autoload.php';

// PayPal API Credentials
use PayPal\Rest\ApiContext;
use PayPal\Auth\OAuthTokenCredential;

// Initialize PayPal API Context
$paypal = new ApiContext(
    new OAuthTokenCredential(
        'YOUR_PAYPAL_CLIENT_ID',
        'YOUR_PAYPAL_SECRET'
    )
);
$paypal->setConfig([
    'mode' => 'sandbox', // Change to 'live' in production
    'log.LogEnabled' => true,
    'log.FileName' => '../PayPal.log',
    'log.LogLevel' => 'FINE'
]);

// Stripe API Key
\Stripe\Stripe::setApiKey('YOUR_STRIPE_SECRET_KEY');

// Process Payment
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $paymentMethod = $_POST['payment_method']; // 'paypal' or 'credit_card'
    $amount = (float)$_POST['amount'];
    $currency = 'USD'; // Change currency as required

    try {
        if ($paymentMethod === 'paypal') {
            // Create PayPal payment
            $payment = new \PayPal\Api\Payment();
            $payment->setIntent("sale")
                ->setPayer(new \PayPal\Api\Payer(['payment_method' => 'paypal']))
                ->setTransactions([
                    new \PayPal\Api\Transaction([
                        'amount' => new \PayPal\Api\Amount([
                            'total' => $amount,
                            'currency' => $currency
                        ]),
                        'description' => "IPTV Subscription"
                    ])
                ])
                ->setRedirectUrls(new \PayPal\Api\RedirectUrls([
                    'return_url' => 'https://yourdomain.com/payment-success',
                    'cancel_url' => 'https://yourdomain.com/payment-cancel'
                ]));
            $payment->create($paypal);
            header("Location: " . $payment->getApprovalLink());
            exit;
        } elseif ($paymentMethod === 'credit_card') {
            // Process Stripe payment
            $charge = \Stripe\Charge::create([
                'amount' => $amount * 100, // Stripe expects amount in cents
                'currency' => $currency,
                'source' => $_POST['stripe_token'],
                'description' => "IPTV Subscription"
            ]);
            echo "Payment Successful. Charge ID: " . $charge->id;
        }
    } catch (Exception $e) {
        echo "Payment failed: " . $e->getMessage();
    }
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <title>IPTV Payment</title>
</head>
<body>
    <h1>Subscribe to IPTV</h1>
    <form method="POST">
        <label for="amount">Amount (USD):</label>
        <input type="number" id="amount" name="amount" required step="0.01" min="0">
        <br><br>
        <label for="payment_method">Payment Method:</label>
        <select id="payment_method" name="payment_method" required>
            <option value="paypal">PayPal</option>
            <option value="credit_card">Credit Card</option>
        </select>
        <br><br>
        <div id="stripe_section" style="display: none;">
            <label for="stripe_token">Credit Card Token:</label>
            <input type="text" id="stripe_token" name="stripe_token">
        </div>
        <br>
        <button type="submit">Pay Now</button>
    </form>

    <script>
        document.getElementById('payment_method').addEventListener('change', function () {
            const stripeSection = document.getElementById('stripe_section');
            stripeSection.style.display = this.value === 'credit_card' ? 'block' : 'none';
        });
    </script>
</body>
</html>
