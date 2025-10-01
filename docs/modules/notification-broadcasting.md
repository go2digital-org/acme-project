# Real-time Notification Broadcasting System

This module implements a comprehensive WebSocket broadcasting integration for the ACME Corp CSR Platform notification system, providing real-time delivery of notifications through Laravel Echo and broadcasting channels.

## Overview

The broadcasting system enables real-time notifications for:
- Large donation alerts
- Campaign milestone achievements  
- Campaign approval requests
- Payment failures
- Security alerts
- System maintenance notifications
- Compliance issues
- General user notifications

## Architecture

### Components

1. **Broadcasting Events** (`Events/`)
   - `NotificationBroadcast` - Generic notification broadcasting
   - `DonationNotificationBroadcast` - Donation-specific events
   - `CampaignNotificationBroadcast` - Campaign-specific events
   - `SystemNotificationBroadcast` - System-wide notifications

2. **Services** (`Services/`)
   - `NotificationBroadcaster` - Main broadcasting service with helper methods

3. **Event Listeners** (`Listeners/`)
   - `NotificationBroadcastListener` - Handles notification created events
   - `DonationBroadcastListener` - Handles donation domain events
   - `CampaignBroadcastListener` - Handles campaign domain events

4. **Commands** (`Commands/`)
   - `TestNotificationBroadcastCommand` - Test broadcasting functionality
   - `BroadcastingDebugCommand` - Debug configuration and connectivity
   - `BroadcastingStatusCommand` - Show system status

5. **Service Providers**
   - `BroadcastingServiceProvider` - Registers services and listeners
   - `BroadcastingCommandsProvider` - Registers Artisan commands
   - `BroadcastAuthServiceProvider` - Handles WebSocket authentication

## Channel Architecture

### Public Channels
- `admin-dashboard` - General admin notifications
- `admin-role-{role}` - Role-specific admin notifications
- `security-alerts` - Security alerts (super admins only)
- `system-maintenance` - System maintenance notifications
- `compliance-notifications` - Compliance issues
- `payment-notifications` - Payment-related alerts

### Private Channels
- `user.notifications.{userId}` - Personal user notifications
- `organization.{organizationId}` - Organization-specific notifications
- `campaign.{campaignId}` - Campaign-specific notifications

## Configuration

### Environment Variables

```bash
# Broadcasting Configuration
BROADCAST_CONNECTION=log                # Development: log, Production: pusher
PUSHER_APP_ID=your_app_id
PUSHER_APP_KEY=your_app_key
PUSHER_APP_SECRET=your_app_secret
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=mt1
```

### Laravel Echo Frontend Integration

The system integrates with the existing `useNotificationWebSocket` composable in Vue.js:

```javascript
// User-specific notifications
window.Echo.private(`user.notifications.${userId}`)
  .listen('.notification.created', handlePersonalNotification)

// Admin dashboard notifications  
window.Echo.channel('admin-dashboard')
  .listen('.donation.large', handleLargeDonationNotification)
  .listen('.campaign.milestone', handleCampaignMilestoneNotification)
  .listen('.campaign.approval_needed', handleApprovalNeededNotification)
```

## Usage

### Broadcasting Notifications

```php
use Modules\Notification\Infrastructure\Broadcasting\Services\NotificationBroadcaster;

// Inject the broadcaster service
public function __construct(private NotificationBroadcaster $broadcaster) {}

// Broadcast a large donation alert
$this->broadcaster->broadcastLargeDonationAlert($donation, $threshold);

// Broadcast campaign milestone
$this->broadcaster->broadcastCampaignMilestone($campaign, '50', $milestoneAmount);

// Broadcast security alert
$this->broadcaster->broadcastSecurityAlert('unauthorized_access', $message, 'high', $details);

// Broadcast system maintenance
$this->broadcaster->broadcastMaintenanceNotification('scheduled', $scheduledFor, 120, ['api', 'dashboard']);
```

### Testing Commands

```bash
# Check system status
php artisan notifications:broadcasting-status

# Debug configuration
php artisan notifications:debug-broadcast --show-config --check-env

# Test broadcasting
php artisan notifications:test-broadcast
php artisan notifications:test-broadcast --type=donation --count=3
php artisan notifications:test-broadcast --type=campaign
php artisan notifications:test-broadcast --type=security
php artisan notifications:test-broadcast --type=maintenance
```

## Event Integration

The system automatically listens for domain events and broadcasts appropriate notifications:

- `LargeDonationReceivedEvent` → Broadcasts large donation alert
- `CampaignMilestoneEvent` → Broadcasts milestone achievement
- `CampaignApprovalNeededEvent` → Broadcasts approval request
- `PaymentFailedEvent` → Broadcasts payment failure
- `NotificationCreatedEvent` → Broadcasts new notification

## Security

### Channel Authorization

Private channels require authentication:

```php
// User can only listen to their own notifications
Broadcast::channel('user.notifications.{userId}', function (User $user, int $userId) {
    return $user->id === $userId;
});

// Organization members and admins can listen to org notifications
Broadcast::channel('organization.{organizationId}', function (User $user, int $organizationId) {
    return $user->organization_id === $organizationId || $user->hasAnyRole(['super_admin', 'csr_admin']);
});
```

### Role-based Access

Admin channels verify user roles:

```php
Broadcast::channel('admin-role-{role}', function (User $user, string $role) {
    $adminRoles = ['super_admin', 'csr_admin', 'finance_admin', 'hr_manager'];
    return in_array($role, $adminRoles) && $user->hasRole($role);
});
```

## Frontend Integration

### Vue.js Composable

The `useNotificationWebSocket` composable provides:
- Automatic Laravel Echo connection management
- Channel subscription based on user role
- Real-time notification handling
- Connection status monitoring
- Automatic reconnection logic

### Admin Dashboard Integration

Works with existing `admin-notifications.js` for:
- Real-time dashboard metric updates
- Live campaign progress updates
- Instant approval badge updates
- Security alert visual indicators

## Performance Considerations

- Uses queued broadcasting (`broadcasts` queue)
- Priority queuing for critical alerts (`critical-broadcasts` queue)
- Efficient channel subscription based on user permissions
- Automatic cleanup on disconnection
- Exponential backoff reconnection strategy

## Development vs Production

### Development Mode
- Uses `log` broadcasting driver
- Events logged to `storage/logs/laravel.log`
- No external WebSocket service required
- Perfect for testing event flow

### Production Mode
- Uses `pusher` broadcasting driver
- Requires Pusher account and credentials
- Real WebSocket connections
- Scalable across multiple servers

## Monitoring and Debugging

The system provides comprehensive debugging tools:

```bash
# Show detailed configuration
php artisan notifications:broadcasting-status --detailed --check-deps

# Test connection and channels
php artisan notifications:debug-broadcast --test-connection --test-channels

# Monitor broadcast events in logs
tail -f storage/logs/laravel.log | grep "broadcasted\|Echo"
```

## Error Handling

- Graceful fallback when Echo is unavailable
- Retry mechanism for failed broadcasts
- User preference suppression support
- Comprehensive error logging
- Connection recovery handling

## Integration with Existing Notification System

This broadcasting system seamlessly integrates with the existing notification infrastructure:

- Works with existing `Notification` domain model
- Uses existing notification types and priorities
- Respects user notification preferences
- Integrates with notification repository
- Maintains audit trail in database

The broadcasting layer acts as an additional delivery channel alongside email, SMS, and in-app notifications, providing real-time updates without replacing the persistent notification storage system.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved