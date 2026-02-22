# Credits System Event-Driven Architecture Design

**Date**: 2026-02-14
**Author**: Plan Agent a307422
**Purpose**: Design event-driven architecture for Credits System integration with future features

---

## Why Event-Driven Architecture?

### Problem Statement

积分系统与未来功能强相关（发帖、回复、上传附件、签到等），如果直接依赖其他模块会导致：

1. **紧耦合**：积分系统需要知道帖子、附件等模块的内部逻辑
2. **难以扩展**：每次添加新功能都需要修改积分系统代码
3. **测试困难**：需要模拟多个模块的交互
4. **代码重复**：不同模块可能重复触发积分逻辑

### Solution: Event-Driven Architecture

```
传统方式（紧耦合）:
发帖模块 → 直接调用 CreditsService → 发放积分
回复模块 → 直接调用 CreditsService → 发放积分
上传模块 → 直接调用 CreditsService → 发放积分

事件驱动方式（解耦）:
发帖模块 → 触发 UserPostedEvent → EventDispatcher → CreditsListener → 发放积分
回复模块 → 触发 UserRepliedEvent → EventDispatcher → CreditsListener → 发放积分
上传模块 → 触发 UserUploadedEvent → EventDispatcher → CreditsListener → 发放积分
```

**优势**：
- ✅ **解耦**：积分系统不需要知道其他模块的存在
- ✅ **可扩展**：新功能只需触发事件，无需修改积分系统
- ✅ **可测试**：可以独立测试事件和监听器
- ✅ **可配置**：积分规则可以通过配置文件调整

---

## Architecture Overview

### Core Components

```
┌──────────────────────────────────────────────────────────┐
│                     业务模块                          │
│  (发帖、回复、上传、注册、登录等)                  │
└─────────────────┬────────────────────────────────────────┘
                  │ dispatch(event)
                  ▼
┌──────────────────────────────────────────────────────────┐
│                  EventDispatcher                       │
│          (事件调度器 - 单例模式)                       │
│                                                          │
│  - subscribe(eventName, listener)   订阅事件            │
│  - dispatch(event)                触发事件              │
│  - 支持优先级排序                  │
│  - 支持事件过滤                  │
└─────────────────┬────────────────────────────────────────┘
                  │ notify()
                  ▼
┌──────────────────────────────────────────────────────────┐
│                   Event Listeners                        │
│  ┌──────────────────┬──────────────────┬────────────┐│
│  │ CreditReward    │ CreditLogging   │ CustomListener││
│  │ Listener        │ Listener        │              ││
│  │ (积分奖励)      │ (事件日志)      │ (自定义)     ││
│  └──────────────────┴──────────────────┴────────────┘│
└─────────────────┬────────────────────────────────────────┘
                  │ handle()
                  ▼
┌──────────────────────────────────────────────────────────┐
│                   CreditsService                         │
│              (积分服务 - 执行动作)                        │
└──────────────────────────────────────────────────────────┘
```

---

## 1. Event System Core

### 1.1 CreditEventInterface

**File**: `app/Credits/Events/CreditEventInterface.php`

所有积分相关事件必须实现此接口。

```php
interface CreditEventInterface
{
    /**
     * 获取事件名称
     * @return string 例如: 'user.registered', 'user.posted'
     */
    public function getEventName(): string;

    /**
     * 获取用户ID
     * @return int
     */
    public function getUserId(): int;

    /**
     * 获取事件时间戳
     * @return int Unix timestamp
     */
    public function getTimestamp(): int;

    /**
     * 获取事件数据（数组格式）
     * @return array<string, mixed>
     */
    public function toArray(): array;
}
```

### 1.2 BaseCreditEvent (Abstract Base Class)

**File**: `app/Credits/Events/BaseCreditEvent.php`

提供通用功能的基础事件类。

```php
readonly class BaseCreditEvent implements CreditEventInterface
{
    public int $userId;
    public int $timestamp;
    public array $metadata = [];

    public function __construct(int $userId, array $metadata = [])
    {
        if ($userId <= 0) {
            throw new \InvalidArgumentException('User ID must be positive');
        }
        $this->userId = $userId;
        $this->timestamp = time();
        $this->metadata = $metadata;
    }

    public function getUserId(): int
    {
        return $this->userId;
    }

    public function getTimestamp(): int
    {
        return $this->timestamp;
    }

    public function getMetadata(string $key, mixed $default = null): mixed
    {
        return $this->metadata[$key] ?? $default;
    }

    public function hasMetadata(string $key): bool
    {
        return isset($this->metadata[$key]);
    }

    public function toArray(): array
    {
        return [
            'event_name' => $this->getEventName(),
            'user_id' => $this->userId,
            'timestamp' => $this->timestamp,
            'metadata' => $this->metadata,
        ];
    }
}
```

---

## 2. Concrete Event Classes

### 2.1 UserRegisteredEvent

**触发时机**: 用户注册成功后

```php
readonly class UserRegisteredEvent extends BaseCreditEvent
{
    public string $username;
    public string $email;
    public int $groupId = 10;

    public function __construct(
        int $userId,
        string $username,
        string $email,
        int $groupId = 10,
        array $metadata = []
    ) {
        $this->username = $username;
        $this->email = $email;
        $this->groupId = $groupId;
        parent::__construct($userId, $metadata);
    }

    public function getEventName(): string
    {
        return 'user.registered';
    }

    public function toArray(): array
    {
        return array_merge(parent::toArray(), [
            'username' => $this->username,
            'email' => $this->email,
            'group_id' => $this->groupId,
        ]);
    }
}
```

**默认积分奖励**: extcredits1 +100, extcredits2 +50

### 2.2 UserPostedEvent

**触发时机**: 用户发帖成功后

```php
readonly class UserPostedEvent extends BaseCreditEvent
{
    public int $postId;
    public int $forumId;
    public string $title;
    public int $length = 0;

    public function __construct(
        int $userId,
        int $postId,
        int $forumId,
        string $title,
        int $length = 0,
        array $metadata = []
    ) {
        $this->postId = $postId;
        $this->forumId = $forumId;
        $this->title = $title;
        $this->length = $length;
        parent::__construct($userId, $metadata);
    }

    public function getEventName(): string
    {
        return 'user.posted';
    }

    public function toArray(): array
    {
        return array_merge(parent::toArray(), [
            'post_id' => $this->postId,
            'forum_id' => $this->forumId,
            'title' => $this->title,
            'length' => $this->length,
        ]);
    }
}
```

**默认积分奖励**: extcredits1 +5, extcredits2 +2

### 2.3 UserRepliedEvent

**触发时机**: 用户回复帖子后

```php
readonly class UserRepliedEvent extends BaseCreditEvent
{
    public int $replyId;
    public int $threadId;
    public int $forumId;
    public int $length = 0;

    public function __construct(
        int $userId,
        int $replyId,
        int $threadId,
        int $forumId,
        int $length = 0,
        array $metadata = []
    ) {
        $this->replyId = $replyId;
        $this->threadId = $threadId;
        $this->forumId = $forumId;
        $this->length = $length;
        parent::__construct($userId, $metadata);
    }

    public function getEventName(): string
    {
        return 'user.replied';
    }

    public function toArray(): array
    {
        return array_merge(parent::toArray(), [
            'reply_id' => $this->replyId,
            'thread_id' => $this->threadId,
            'forum_id' => $this->forumId,
            'length' => $this->length,
        ]);
    }
}
```

**默认积分奖励**: extcredits1 +2, extcredits2 +1

### 2.4 UserUploadedEvent

**触发时机**: 用户上传附件后

```php
readonly class UserUploadedEvent extends BaseCreditEvent
{
    public int $attachmentId;
    public string $fileName;
    public int $fileSize;
    public string $fileType;

    public function __construct(
        int $userId,
        int $attachmentId,
        string $fileName,
        int $fileSize,
        string $fileType,
        array $metadata = []
    ) {
        $this->attachmentId = $attachmentId;
        $this->fileName = $fileName;
        $this->fileSize = $fileSize;
        $this->fileType = $fileType;
        parent::__construct($userId, $metadata);
    }

    public function getEventName(): string
    {
        return 'user.uploaded';
    }

    public function toArray(): array
    {
        return array_merge(parent::toArray(), [
            'attachment_id' => $this->attachmentId,
            'file_name' => $this->fileName,
            'file_size' => $this->fileSize,
            'file_type' => $this->fileType,
        ]);
    }
}
```

**默认积分奖励**: extcredits1 +3, extcredits3 +1

### 2.5 UserLoggedInEvent

**触发时机**: 用户登录成功后

```php
readonly class UserLoggedInEvent extends BaseCreditEvent
{
    public string $loginMethod = 'web';
    public string $ipAddress = '0.0.0.0';
    public ?string $userAgent = null;

    public function __construct(
        int $userId,
        string $loginMethod = 'web',
        string $ipAddress = '0.0.0.0',
        ?string $userAgent = null,
        array $metadata = []
    ) {
        $this->loginMethod = $loginMethod;
        $this->ipAddress = $ipAddress;
        $this->userAgent = $userAgent;
        parent::__construct($userId, $metadata);
    }

    public function getEventName(): string
    {
        return 'user.logged_in';
    }

    public function toArray(): array
    {
        return array_merge(parent::toArray(), [
            'login_method' => $this->loginMethod,
            'ip_address' => $this->ipAddress,
            'user_agent' => $this->userAgent,
        ]);
    }
}
```

**默认积分奖励**: extcredits1 +1（每日限制）

---

## 3. Event Dispatcher

### 3.1 EventDispatcherInterface

**File**: `app/Credits/Events/EventDispatcherInterface.php`

```php
interface EventDispatcherInterface
{
    /**
     * 订阅事件
     * @param string $eventName 事件名称
     * @param EventListenerInterface $listener 事件监听器
     */
    public function subscribe(string $eventName, EventListenerInterface $listener): void;

    /**
     * 触发事件
     * @param CreditEventInterface $event 事件对象
     */
    public function dispatch(CreditEventInterface $event): void;

    /**
     * 取消订阅
     * @param string $eventName 事件名称
     * @param EventListenerInterface $listener 事件监听器
     * @return bool 是否成功取消
     */
    public function unsubscribe(string $eventName, EventListenerInterface $listener): bool;
}
```

### 3.2 EventDispatcher Implementation

**File**: `app/Credits/Events/EventDispatcher.php`

核心功能：
- **单例模式**：全局唯一实例
- **优先级支持**：高优先级监听器先执行
- **事件过滤**：监听器可以决定是否处理事件
- **日志支持**：可启用/禁用日志
- **错误处理**：监听器异常会停止传播

```php
class EventDispatcher
{
    private static ?EventDispatcher $instance = null;
    private array $listeners = [];
    private LoggerInterface $logger;
    private bool $enableLogging = false;

    private function __construct(?LoggerInterface $logger = null, bool $enableLogging = false)
    {
        $this->logger = $logger ?? new NullLogger();
        $this->enableLogging = $enableLogging;
    }

    public static function getInstance(
        ?LoggerInterface $logger = null,
        bool $enableLogging = false
    ): self {
        if (self::$instance === null) {
            self::$instance = new self($logger, $enableLogging);
        }
        return self::$instance;
    }

    public function subscribe(string $eventName, EventListenerInterface $listener): void
    {
        if (!isset($this->listeners[$eventName])) {
            $this->listeners[$eventName] = [];
        }

        $this->listeners[$eventName][] = [
            'listener' => $listener,
            'priority' => $listener->getPriority(),
        ];

        // 按优先级排序（高优先级先执行）
        usort($this->listeners[$eventName], function($a, $b) {
            return $b['priority'] <=> $a['priority'];
        });
    }

    public function dispatch(CreditEventInterface $event): void
    {
        $eventName = $event->getEventName();

        if (!isset($this->listeners[$eventName])) {
            return;
        }

        foreach ($this->listeners[$eventName] as $listenerData) {
            $listener = $listenerData['listener'];

            try {
                // 检查监听器是否应该处理此事件
                if (!$listener->shouldHandle($event)) {
                    continue;
                }

                // 处理事件
                $listener->handle($event);
            } catch (\Throwable $e) {
                // 记录错误并重新抛出（停止传播）
                $this->logger->error("Error in listener: {$e->getMessage()}", [
                    'exception' => $e,
                    'event' => $eventName,
                ]);
                throw $e;
            }
        }
    }

    public function unsubscribe(string $eventName, EventListenerInterface $listener): bool
    {
        if (!isset($this->listeners[$eventName])) {
            return false;
        }

        foreach ($this->listeners[$eventName] as $index => $listenerData) {
            if ($listenerData['listener'] === $listener) {
                unset($this->listeners[$eventName][$index]);
                return true;
            }
        }

        return false;
    }

    // 防止克隆
    private function __clone() {}

    // 防止反序列化
    public function __wakeup(): void
    {
        throw new \RuntimeException('Cannot unserialize singleton');
    }
}
```

---

## 4. Event Listener Interface

### 4.1 EventListenerInterface

**File**: `app/Credits/Events/EventListenerInterface.php`

```php
interface EventListenerInterface
{
    /**
     * 处理事件
     * @param CreditEventInterface $event
     */
    public function handle(CreditEventInterface $event): void;

    /**
     * 检查是否应该处理此事件
     * @param CreditEventInterface $event
     * @return bool true = 处理, false = 跳过
     */
    public function shouldHandle(CreditEventInterface $event): bool;

    /**
     * 获取监听器优先级
     * @return int 优先级（数值越大优先级越高）
     */
    public function getPriority(): int;

    /**
     * 获取监听器名称
     * @return string
     */
    public function getName(): string;
}
```

### 4.2 AbstractEventListener (Base Class)

**File**: `app/Credits/Events/AbstractEventListener.php`

提供通用功能的基类。

```php
abstract class AbstractEventListener implements EventListenerInterface
{
    private int $priority;
    private string $name;

    public function __construct(int $priority = 0)
    {
        $this->priority = $priority;
        $this->name = static::class;
    }

    public function getPriority(): int
    {
        return $this->priority;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }

    public function shouldHandle(CreditEventInterface $event): bool
    {
        return true; // 默认处理所有事件
    }
}
```

---

## 5. Credit Rules Configuration

### 5.1 CreditRules Class

**File**: `app/Credits/Config/CreditRules.php`

管理积分奖励规则。

```php
class CreditRules
{
    private array $rules;
    private Config $config;
    private bool $enableDailyLimits;

    public function __construct(Config $config, array $rules = [])
    {
        $this->config = $config;
        $this->enableDailyLimits = $config->get('credits.daily_limits_enabled', false);
        $this->rules = empty($rules) ? $this->loadDefaultRules() : $rules;
    }

    public function getReward(string $eventName): array
    {
        $default = ['extcredits1' => 0, 'extcredits2' => 0, 'extcredits3' => 0];
        return $this->rules[$eventName] ?? $default;
    }

    public function getRewardAmount(string $eventName, string $creditType): int
    {
        $reward = $this->getReward($eventName);
        return $reward[$creditType] ?? 0;
    }

    public function hasReward(string $eventName): bool
    {
        $reward = $this->getReward($eventName);
        return !empty(array_filter($reward));
    }

    public function getDailyLimit(string $eventName, string $creditType): ?int
    {
        if (!$this->enableDailyLimits) {
            return null;
        }

        $key = "{$eventName}.{$creditType}.daily_limit";
        return $this->config->get("credits.limits.{$key}");
    }

    private function loadDefaultRules(): array
    {
        return [
            'user.registered' => [
                'extcredits1' => 100,  // 注册奖励
                'extcredits2' => 50,
                'extcredits3' => 0,
            ],
            'user.posted' => [
                'extcredits1' => 5,     // 发帖奖励
                'extcredits2' => 2,
                'extcredits3' => 0,
            ],
            'user.replied' => [
                'extcredits1' => 2,     // 回复奖励
                'extcredits2' => 1,
                'extcredits3' => 0,
            ],
            'user.uploaded' => [
                'extcredits1' => 3,     // 上传奖励
                'extcredits2' => 0,
                'extcredits3' => 1,
            ],
            'user.logged_in' => [
                'extcredits1' => 1,     // 每日登录奖励
                'extcredits2' => 0,
                'extcredits3' => 0,
            ],
            'credits.transferred' => [
                'extcredits1' => 0,     // 转账无额外奖励
                'extcredits2' => 0,
                'extcredits3' => 0,
            ],
        ];
    }
}
```

---

## 6. Credit Listeners

### 6.1 CreditRewardListener

**File**: `app/Credits/Listeners/CreditRewardListener.php`

自动根据事件发放积分奖励。

```php
class CreditRewardListener extends AbstractEventListener
{
    private CreditService $creditService;
    private CreditRules $rules;
    private array $dailyRewards = [];
    private LoggerInterface $logger;

    public function __construct(
        CreditService $creditService,
        CreditRules $rules,
        ?LoggerInterface $logger = null,
        int $priority = 10
    ) {
        $this->creditService = $creditService;
        $this->rules = $rules;
        $this->logger = $logger ?? new NullLogger();
        parent::__construct($priority);
    }

    public function handle(CreditEventInterface $event): void
    {
        $eventName = $event->getEventName();
        $userId = $event->getUserId();

        // 获取奖励规则
        $rewards = $this->rules->getReward($eventName);

        // 检查是否有奖励
        if (empty($rewards) || !$this->rules->hasReward($eventName)) {
            $this->logger->debug("No rewards configured for event: {$eventName}");
            return;
        }

        // 检查每日限制
        if (!$this->checkDailyLimits($userId, $eventName, $rewards)) {
            $this->logger->info("Daily limit reached for user: {$userId}, event: {$eventName}");
            return;
        }

        // 为每种积分类型发放奖励
        foreach ($rewards as $creditType => $amount) {
            if ($amount === 0) {
                continue;
            }

            try {
                $operation = $this->getOperationDescription($eventName, $creditType);

                $dto = new CreditTransactionDto(
                    userId: $userId,
                    type: $eventName,
                    amount: $amount,
                    operation: $operation,
                    relatedId: $event->getMetadata('related_id')
                );

                if ($amount > 0) {
                    $this->creditService->credit($dto);
                } else {
                    $this->creditService->debit($dto);
                }

                $this->trackDailyReward($userId, $eventName, $creditType, $amount);

                $this->logger->info("Granted {$amount} {$creditType} to user {$userId} for {$eventName}");

            } catch (\Throwable $e) {
                $this->logger->error("Failed to grant credits: {$e->getMessage()}", [
                    'user_id' => $userId,
                    'event' => $eventName,
                    'credit_type' => $creditType,
                    'amount' => $amount,
                ]);
            }
        }
    }

    private function checkDailyLimits(int $userId, string $eventName, array $rewards): bool
    {
        $today = date('Y-m-d');
        $key = "{$userId}_{$today}";

        foreach ($rewards as $creditType => $amount) {
            $limit = $this->rules->getDailyLimit($eventName, $creditType);

            if ($limit !== null) {
                $current = $this->dailyRewards[$key][$eventName][$creditType] ?? 0;

                if (($current + $amount) > $limit) {
                    return false;
                }
            }
        }

        return true;
    }

    private function trackDailyReward(int $userId, string $eventName, string $creditType, int $amount): void
    {
        $today = date('Y-m-d');
        $key = "{$userId}_{$today}";

        if (!isset($this->dailyRewards[$key][$eventName][$creditType])) {
            $this->dailyRewards[$key][$eventName][$creditType] = 0;
        }

        $this->dailyRewards[$key][$eventName][$creditType] += $amount;
    }

    private function getOperationDescription(string $eventName, string $creditType): string
    {
        $labels = [
            'user.registered' => 'Registration bonus',
            'user.posted' => 'Post reward',
            'user.replied' => 'Reply reward',
            'user.uploaded' => 'Upload reward',
            'user.logged_in' => 'Daily login reward',
            'credits.transferred' => 'Transfer',
        ];

        return $labels[$eventName] ?? 'Event reward';
    }
}
```

---

## 7. Usage Examples

### 7.1 在用户注册时触发事件

**File**: `app/User/Services/UserRegistrationService.php`

```php
public function register(UserRegistrationDto $dto): User
{
    // 1. 验证数据
    // 2. 创建用户
    // 3. 发送验证邮件

    // 4. 触发注册事件（自动发放积分）
    $registrationEvent = new UserRegisteredEvent(
        userId: $user->getUserId(),
        username: $user->getUsername(),
        email: $user->getEmail(),
        groupId: $user->getGroupId()
    );

    $eventDispatcher = \Discuz\Credits\Events\EventDispatcher::getInstance();
    $eventDispatcher->dispatch($registrationEvent);

    return $user;
}
```

### 7.2 在发帖时触发事件

**File**: `app/Forum/Services/PostService.php` (未来实现)

```php
public function createPost(CreatePostDto $dto): Post
{
    // 1. 验证数据
    // 2. 创建帖子
    // 3. 更新论坛统计

    // 4. 触发发帖事件（自动发放积分）
    $postedEvent = new UserPostedEvent(
        userId: $dto->userId,
        postId: $post->getPostId(),
        forumId: $post->getForumId(),
        title: $post->getTitle(),
        length: mb_strlen($post->getMessage())
    );

    $eventDispatcher = \Discuz\Credits\Events\EventDispatcher::getInstance();
    $eventDispatcher->dispatch($postedEvent);

    return $post;
}
```

### 7.3 配置积分规则

**File**: `config/credits.php`

```php
return [
    // 启用每日限制
    'daily_limits_enabled' => true,

    // 积分类型名称
    'credit_names' => [
        'extcredits1' => 'Credits',
        'extcredits2' => 'Points',
        'extcredits3' => 'Gold',
    ],

    // 每日限制
    'limits' => [
        'user.posted.extcredits1.daily_limit' => 50,  // 每日最多从发帖获得 50 extcredits1
        'user.posted.extcredits2.daily_limit' => 20,
        'user.replied.extcredits1.daily_limit' => 20,  // 每日最多从回复获得 20 extcredits1
        'user.logged_in.extcredits1.daily_limit' => 5,  // 每日最多从登录获得 5 extcredits1
    ],

    // 自定义规则（覆盖默认值）
    'rules' => [
        'user.registered' => [
            'extcredits1' => 100,
            'extcredits2' => 50,
        ],
    ],
];
```

---

## 8. Testing

### 8.1 测试事件系统

```php
use PHPUnit\Framework\TestCase;
use Discuz\Credits\Events\EventDispatcher;
use Discuz\Credits\Events\UserRegisteredEvent;
use Discuz\Credits\Listeners\CreditRewardListener;

class EventSystemTest extends TestCase
{
    public function testRegistrationGrantsCredits()
    {
        // 创建测试dispatcher
        $dispatcher = new EventDispatcher();

        // 订阅积分奖励监听器
        $creditListener = new CreditRewardListener(
            $creditService,
            new CreditRules($config)
        );
        $dispatcher->subscribe('user.registered', $creditListener);

        // 触发注册事件
        $event = new UserRegisteredEvent(1, 'testuser', 'test@example.com');
        $dispatcher->dispatch($event);

        // 断言积分已发放
        $credits = $creditService->getUserCredits(1);
        $this->assertEquals(100, $credits['extcredits1']);
        $this->assertEquals(50, $credits['extcredits2']);
    }
}
```

---

## 9. Benefits of Event-Driven Architecture

### 9.1 解耦

**之前**：
```php
// 发帖模块直接依赖积分系统
class PostService {
    public function createPost($dto) {
        $post = $this->savePost($dto);

        // 直接调用积分服务
        $this->creditService->grantPostReward($post->getUserId(), $post->getPostId());

        return $post;
    }
}
```

**现在**：
```php
// 发帖模块只触发事件，不知道积分系统的存在
class PostService {
    public function createPost($dto) {
        $post = $this->savePost($dto);

        // 触发事件
        $event = new UserPostedEvent($post->getUserId(), $post->getPostId(), ...);
        EventDispatcher::getInstance()->dispatch($event);

        return $post;
    }
}
```

### 9.2 可扩展

添加新功能时无需修改积分系统：

```php
// 新功能：每日签到
class CheckInService {
    public function checkIn(int $userId): void {
        // 记录签到

        // 触发事件
        $event = new UserCheckedInEvent($userId);
        EventDispatcher::getInstance()->dispatch($event);
    }
}

// 只需在配置文件添加规则
'rules' => [
    'user.checked_in' => [
        'extcredits1' => 10,  // 每日签到 +10 积分
    ],
],
```

### 9.3 可配置

积分规则可以在配置文件中调整，无需修改代码：

```php
// config/credits.php

// 节日模式：双倍积分
if (date('Y-m-d') === '2026-02-14') {
    return [
        'rules' => [
            'user.posted' => [
                'extcredits1' => 10,  // 双倍！
                'extcredits2' => 4,  // 双倍！
            ],
        ],
    ];
}
```

---

## 10. Migration Guide

### 10.1 从直接调用迁移到事件驱动

**旧代码**（直接调用）：
```php
// 在用户注册时直接发放积分
class UserRegistrationService {
    private CreditService $creditService;

    public function register($dto) {
        $user = $this->createUser($dto);

        // 直接发放积分
        $this->creditService->credit(
            new CreditTransactionDto(
                userId: $user->getId(),
                type: 'registration',
                amount: 100,
                operation: 'Registration bonus'
            )
        );

        return $user;
    }
}
```

**新代码**（事件驱动）：
```php
// 在用户注册时触发事件
class UserRegistrationService {
    public function register($dto) {
        $user = $this->createUser($dto);

        // 触发注册事件
        $event = new UserRegisteredEvent(
            userId: $user->getId(),
            username: $user->getUsername(),
            email: $user->getEmail()
        );
        EventDispatcher::getInstance()->dispatch($event);

        return $user;
    }
}

// CreditRewardListener 会自动处理事件并发放积分
```

**优势**：
- ✅ UserRegistrationService 不再依赖 CreditService
- ✅ 可以添加其他监听器（例如：发送欢迎邮件、记录日志等）
- ✅ 积分规则可以在配置文件中调整
- ✅ 更容易测试（可以 mock EventDispatcher）

---

## 11. Implementation Priority

### Phase 1: Core Event System (当前优先)
1. ✅ CreditEventInterface
2. ✅ BaseCreditEvent
3. ✅ EventListenerInterface
4. ✅ AbstractEventListener
5. ✅ EventDispatcher
6. ✅ Concrete Event Classes (6+ events)

### Phase 2: Credit Integration (Day 15 - 当前)
1. ⏳ CreditRules Configuration
2. ⏳ CreditRewardListener
3. ⏳ CreditLoggingListener
4. ⏳ Integration with UserRegistrationService
5. ⏳ Documentation

### Phase 3: Future Features (Week 4+)
1. ⏳ Post System Integration
2. ⏳ Reply System Integration
3. ⏳ Attachment System Integration
4. ⏳ Check-in Feature
5. ⏳ Other custom events

---

## 12. Files to Create

### Core Event System
- `app/Credits/Events/CreditEventInterface.php`
- `app/Credits/Events/BaseCreditEvent.php`
- `app/Credits/Events/EventListenerInterface.php`
- `app/Credits/Events/AbstractEventListener.php`
- `app/Credits/Events/EventDispatcher.php`

### Concrete Events
- `app/Credits/Events/UserRegisteredEvent.php`
- `app/Credits/Events/UserPostedEvent.php`
- `app/Credits/Events/UserRepliedEvent.php`
- `app/Credits/Events/UserUploadedEvent.php`
- `app/Credits/Events/UserLoggedInEvent.php`
- `app/Credits/Events/CreditsTransferredEvent.php`

### Configuration
- `app/Credits/Config/CreditRules.php`
- `config/credits.php`

### Listeners
- `app/Credits/Listeners/CreditRewardListener.php`
- `app/Credits/Listeners/CreditLoggingListener.php`

### Tests
- `tests/Unit/Credits/Events/*Test.php`
- `tests/Unit/Credits/Listeners/*Test.php`

---

**生成时间**: 2026-02-14
**设计者**: Plan Agent a307422
**状态**: ✅ 设计完成，待实施
