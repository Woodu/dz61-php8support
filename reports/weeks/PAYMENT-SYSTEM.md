# Thread Payment System Documentation

## Overview

The Thread Payment system allows users to charge credits for viewing their thread content. Users must pay to view the full content, unless they are the thread author or have already paid.

## Features

- **Pay-to-view threads**: Set a price on threads to earn credits
- **Free preview content**: Use `[free]...[/free]` tags to show free previews
- **Automatic tax calculation**: Configurable tax rate applied to payments
- **Income tracking**: Track total income and payer count per thread
- **Max income limit**: Optionally cap maximum income per thread
- **Charging duration**: Optionally limit how long threads can charge

## Database Schema

### cdb_paymentlog
```sql
CREATE TABLE cdb_paymentlog (
  uid mediumint unsigned NOT NULL DEFAULT '0',
  tid mediumint unsigned NOT NULL DEFAULT '0',
  authorid mediumint unsigned NOT NULL DEFAULT '0',
  dateline int unsigned NOT NULL DEFAULT '0',
  amount int unsigned NOT NULL DEFAULT '0',
  netamount int unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (tid, uid),
  KEY uid (uid),
  KEY authorid (authorid)
) ENGINE=MyISAM DEFAULT CHARSET=utf8mb4;
```

### cdb_threads.price
- Field: `price` (smallint)
- Description: Price in credits (0 = free thread)

## Configuration

Settings are stored in `cdb_settings`:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `creditstax` | float | 0.01 | Tax rate (0.01 = 1%) |
| `creditstrans` | int | 2 | Credit type to use (1-8) |
| `maxincperthread` | int | 0 | Max income per thread (0 = unlimited) |
| `maxchargespan` | int | 0 | Max charging duration in hours (0 = unlimited) |

## Usage Examples

### 1. Check if User Needs to Pay

```php
use Discuz\Payment\Services\PaymentService;

$paymentService = new PaymentService(
    $paymentRepository,
    $creditService,
    $postRepository,
    $db
);

$requiresPayment = $paymentService->requiresPayment(
    threadId: 123,
    userId: 456,
    authorId: 789
);

if ($requiresPayment) {
    // Show payment prompt
}
```

### 2. Process Payment

```php
use Discuz\Payment\DTOs\PayDto;

$dto = new PayDto(
    threadId: 123,
    userId: 456,
    authorId: 789,
    amount: 100,
    creditType: 'extcredits2',
    ipAddress: $_SERVER['REMOTE_ADDR']
);

try {
    $paymentService->payForThread($dto);
    echo "Payment successful! You can now view the thread.";
} catch (PaymentException $e) {
    echo "Payment failed: " . $e->getMessage();
}
```

### 3. Get Payment Status for Thread View

```php
$paymentView = $paymentService->getPaymentStatus(
    threadId: 123,
    userId: 456
);

if ($paymentView) {
    echo "Price: " . $paymentView->price . " credits\n";
    echo "Payers: " . $paymentView->payerCount . "\n";
    echo "Total Income: " . $paymentView->totalIncome . "\n";
    echo "Has Paid: " . ($paymentView->hasPaid ? 'Yes' : 'No') . "\n";
    echo "Requires Payment: " . ($paymentView->requiresPayment ? 'Yes' : 'No') . "\n";

    if (!$paymentView->canViewFullContent()) {
        echo "Free Preview:\n" . $paymentView->freeContent;
    }
}
```

### 4. Extract Free Content

```php
// For non-paying users, show only free content
$message = $post->getMessage();

if ($paymentView->requiresPayment && !$paymentView->hasPaid) {
    $freeContent = $paymentService->extractFreeContentFromMessage(
        $message,
        hasPaid: false
    );
    echo $freeContent . "\n\n[Purchase full content for " . $paymentView->price . " credits]";
} else {
    // Show full content
    echo $message;
}
```

### 5. Create Thread with Price

```php
// Set thread price when creating
$threadData = [
    'fid' => 1,
    'subject' => 'Premium Thread',
    'authorid' => 123,
    'price' => 100, // Price in credits
    // ... other fields
];

$threadRepository->create($threadData);
```

### 6. Display Thread with Payment Info (in ThreadViewService)

```php
// ThreadViewService automatically includes payment data
$threadView = $threadViewService->getThreadWithPosts(
    threadId: 123,
    userId: 456
);

$paymentView = $threadView->getPaymentView();
if ($paymentView) {
    // Display payment information
    if ($paymentView->requiresPayment && !$paymentView->canViewFullContent()) {
        echo "Price: " . $paymentView->price . " credits";
        echo $paymentView->freeContent;
    }
}
```

## Free Content Tags

Use `[free]...[/free]` tags in post messages to show content without payment:

```text
This is hidden content.

[free]
This content is visible to everyone, even without payment!
You can use this to provide a preview.
[/free]

More hidden content here.
```

### Example Output

**For non-paying users:**
```
This content is visible to everyone, even without payment!
You can use this to provide a preview.

[Purchase full content for 100 credits]
```

**For paying users:**
```
This is hidden content.

This content is visible to everyone, even without payment!
You can use this to provide a preview.

More hidden content here.
```

## Payment Flow

1. **User visits thread**: Check `requiresPayment()`
2. **Show payment prompt**: If required and not paid
3. **Process payment**: Call `payForThread()`
   - Debits credits from user
   - Credits net amount (minus tax) to author
   - Records payment in `cdb_paymentlog`
4. **Grant access**: User can now view full content

## Tax Calculation

```
Net Amount = Amount × (1 - tax_rate)

Example:
- Price: 100 credits
- Tax rate: 1% (0.01)
- Net amount: 100 × (1 - 0.01) = 99 credits
```

## Income Limits

If `maxincperthread` is set (> 0):

- Once total income reaches the limit, `canCharge` becomes `false`
- Thread becomes free for all users
- Author no longer receives payments

Example:
```
maxincperthread = 1000
price = 100
→ After 10 payments (1000 income), thread becomes free
```

## Time Limits

If `maxchargespan` is set (> 0):

- Charging automatically stops after the duration
- Thread becomes free for all users after expiration

Example:
```
maxchargespan = 24 (hours)
→ Thread can only charge for 24 hours from creation
```

## API Reference

### PaymentService

```php
// Check if payment required
public function requiresPayment(int $threadId, int $userId, int $authorId): bool

// Process payment
public function payForThread(PayDto $dto): void

// Get payment status
public function getPaymentStatus(int $threadId, int $userId): ?PaymentView

// Extract free content
public function extractFreeContentFromMessage(string $message, bool $hasPaid): string

// Calculate net amount
public function calculateNetAmount(int $amount, ?float $taxRate = null): int

// Get thread pricing info
public function getThreadPrice(int $threadId): ThreadPrice
```

### PaymentRepository

```php
// Get payment log
public function getPaymentLog(int $userId, int $threadId): ?PaymentLog

// Check if paid
public function hasPaid(int $userId, int $threadId): bool

// Get thread stats
public function getThreadPaymentStats(int $threadId): array

// Get thread price
public function getThreadPrice(int $threadId): int

// Get thread author
public function getThreadAuthorId(int $threadId): int

// Create payment log
public function createPaymentLog(PaymentLog $log): void
```

### PaymentView DTO

```php
readonly class PaymentView
{
    public int $threadId;
    public int $price;
    public int $netPrice;
    public int $payerCount;
    public int $totalIncome;
    public float $taxRate;
    public string $taxRateFormatted;
    public bool $hasPaid;
    public bool $isAuthor;
    public bool $requiresPayment;
    public bool $canCharge;
    public ?int $endTime;
    public int $maxIncome;
    public string $freeContent;

    public function canViewFullContent(): bool
    public function getDisplayPrice(): int
}
```

## Error Handling

All payment exceptions extend `PaymentException`:

```php
use Discuz\Payment\Exceptions\PaymentException;

try {
    $paymentService->payForThread($dto);
} catch (PaymentException $e) {
    // Handle specific payment errors
    if (str_contains($e->getMessage(), 'Insufficient credits')) {
        echo "You don't have enough credits.";
    } elseif (str_contains($e->getMessage(), 'already paid')) {
        echo "You've already paid for this thread.";
    }
}
```

## Testing

Run payment tests:

```bash
docker exec -i discuz_modern_php vendor/bin/phpunit tests/Unit/Payment/
```

## Integration with ThreadViewService

The payment system integrates with `ThreadViewService`:

```php
use Discuz\Thread\Services\ThreadViewService;
use Discuz\Payment\Services\PaymentService;

$threadViewService = new ThreadViewService(
    $threadRepository,
    $postRepository,
    $cache,
    $pollService,
    $paymentService // Optional payment service
);

$threadView = $threadViewService->getThreadWithPosts(123, 456);
$paymentView = $threadView->getPaymentView();
```

## Legacy Compatibility

This system replaces `include/threadpay.inc.php` from Discuz! 6.1F:

**Legacy features preserved:**
- Payment logging in `cdb_paymentlog`
- Thread pricing in `cdb_threads.price`
- Tax calculation from settings
- Free content preview with `[free]` tags
- Income and payer tracking
- Max income and charging duration limits

**Modern improvements:**
- PSR-4 autoloading
- PHP 8.3 strict types
- PDO prepared statements
- Service layer architecture
- Comprehensive unit tests
- DTOs for type safety
