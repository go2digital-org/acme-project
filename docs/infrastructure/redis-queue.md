# Redis Queue Infrastructure Implementation

This document provides a comprehensive overview of the Redis queue infrastructure and background job system implemented for the ACME Corp CSR platform.

## Overview

The platform now uses Redis as the primary queue driver with multiple specialized queue channels, comprehensive error handling, progress tracking, and monitoring capabilities. All heavy operations have been moved to background processing to ensure optimal performance for 20,000+ concurrent users.

## Queue Infrastructure

### Redis Configuration

**File**: `/config/database.php`
- Added dedicated Redis `queue` connection on database 7
- Configured with persistent connections and optimized timeouts

**File**: `/config/queue.php`
- **Default Driver**: Redis
- **Multiple Queue Channels**: payments, notifications, exports, reports, maintenance, bulk
- **Priority System**: 1-10 priority levels with dedicated worker processes
- **Retry Logic**: Exponential backoff with configurable attempts
- **Worker Configuration**: Auto-scaling based on queue load and time

### Queue Channels

1. **payments** (Priority 10)
   - Timeout: 180 seconds
   - Max tries: 5
   - Memory: 256MB
   - Exponential backoff: [30, 60, 120, 300, 600] seconds

2. **notifications** (Priority 8)
   - Timeout: 120 seconds
   - Max tries: 3
   - Memory: 128MB
   - Backoff: [15, 30, 60] seconds

3. **exports** (Priority 6)
   - Timeout: 900 seconds (15 minutes)
   - Max tries: 2
   - Memory: 1024MB
   - Backoff: [60, 180] seconds

4. **reports** (Priority 6)
   - Timeout: 600 seconds (10 minutes)
   - Max tries: 3
   - Memory: 512MB

5. **maintenance** (Priority 1)
   - Timeout: 1800 seconds (30 minutes)
   - Max tries: 1
   - Memory: 256MB

6. **bulk** (Priority 2)
   - Timeout: 3600 seconds (1 hour)
   - Max tries: 1
   - Memory: 1024MB

## Background Jobs Implemented

### Payment Processing Jobs

#### ProcessPaymentWebhookJob
**Location**: `/modules/Donation/Infrastructure/Laravel/Jobs/ProcessPaymentWebhookJob.php`
- Handles Stripe, Mollie, and PayPal webhook processing asynchronously
- Verifies webhook signatures
- Maps webhook events to appropriate donation actions
- Handles payment success, failure, and refund events
- Queue: `payments`
- Features:
  - Signature validation
  - Event type mapping
  - Transaction ID extraction
  - Error handling with permanent failure tracking

#### ProcessDonationJob
**Location**: `/modules/Donation/Infrastructure/Laravel/Jobs/ProcessDonationJob.php`
- Complete donation workflow processing
- Updates campaign totals
- Applies corporate matching
- Generates audit logs
- Dispatches notifications
- Queue: `payments`
- Features:
  - Database transactions
  - Corporate matching calculation
  - Campaign milestone tracking
  - Notification dispatch

#### SendPaymentConfirmationJob
**Location**: `/modules/Donation/Infrastructure/Laravel/Jobs/SendPaymentConfirmationJob.php`
- Sends payment confirmation emails to donors
- Supports multiple languages
- Handles anonymous donations
- Queue: `notifications`
- Features:
  - Locale-aware email generation
  - Anonymous donation handling
  - Organizer copy notifications
  - Confirmation number generation

#### RefundProcessingJob
**Location**: `/modules/Donation/Infrastructure/Laravel/Jobs/RefundProcessingJob.php`
- Processes payment refunds
- Updates campaign totals
- Handles partial and full refunds
- Processes corporate matching refunds
- Queue: `payments`
- Features:
  - Gateway refund processing
  - Proportional matching refunds
  - Campaign total adjustments
  - Audit trail creation

### Export and Reporting Jobs

#### ExportCampaignsJob
**Location**: `/modules/Shared/Infrastructure/Laravel/Jobs/ExportCampaignsJob.php`
- Exports campaign data with progress tracking
- Supports multiple formats (Excel, CSV, PDF)
- Chunked processing for large datasets
- Real-time progress updates
- Queue: `exports`
- Features:
  - Progress tracking with WebSocket updates
  - Batch processing
  - Format-specific exports
  - Email notifications with attachments

#### ExportDonationsJob
**Location**: `/modules/Shared/Infrastructure/Laravel/Jobs/ExportDonationsJob.php`
- Exports donation data with privacy controls
- Includes/excludes personal information based on permissions
- Supports tax receipt data export
- Currency breakdown summaries
- Queue: `exports`
- Features:
  - Privacy-aware data export
  - Tax receipt information
  - Currency summaries
  - Large dataset handling

### Communication Jobs

#### SendEmailJob
**Location**: `/modules/Shared/Infrastructure/Laravel/Jobs/SendEmailJob.php`
- Universal email sending job
- Priority-based queue assignment
- Supports multiple recipients
- Attachment handling
- Queue: Dynamic based on priority (notifications/bulk)
- Features:
  - Priority-based processing
  - Multiple recipient types
  - Attachment support
  - Locale awareness
  - Failed email tracking

### Maintenance Jobs

#### CleanupExpiredDataJob
**Location**: `/modules/Shared/Infrastructure/Laravel/Jobs/CleanupExpiredDataJob.php`
- Automated data retention and cleanup
- Configurable retention periods
- Dry-run capability
- Comprehensive reporting
- Queue: `maintenance`
- Features:
  - Job progress cleanup (30 days)
  - Failed jobs cleanup (90 days)
  - Export files cleanup (30 days)
  - Temporary files cleanup (24 hours)
  - Cache and session cleanup
  - Audit log retention (365 days)

## Job Progress Tracking

### JobProgress Model
**Location**: `/modules/Shared/Infrastructure/Laravel/Models/JobProgress.php`
**Migration**: `/modules/Shared/Infrastructure/Laravel/Migration/2025_08_13_200000_create_job_progress_table.php`

Features:
- Real-time progress updates
- Estimated completion time calculations
- Batch processing support
- WebSocket integration for live updates
- Comprehensive metadata storage

### Progress Tracking Capabilities
- Percentage completion tracking
- Item counting (total, processed, failed)
- Time estimation algorithms
- Status management (pending, running, completed, failed, cancelled)
- User association for permission-based access

## Laravel Horizon Integration

### Configuration
**File**: `/config/horizon.php`

Features:
- Environment-specific worker configuration
- Auto-scaling strategies (time and size-based)
- Queue wait time monitoring
- Job trimming policies
- Memory limit management
- Dark mode UI

### Worker Configurations

**Production Environment**:
- High Priority Supervisor: 2-8 processes for payments/notifications
- Exports Supervisor: 2 processes with 1024MB memory
- Reports Supervisor: 2 processes with 512MB memory
- Bulk/Maintenance Supervisors: 1 process each

**Development Environment**:
- Single supervisor handling all queues
- 3 processes with reduced memory limits
- Simplified configuration for local development

## Webhook Processing Updates

### Updated Controllers

#### StripeWebhookController
**Location**: `/modules/Donation/Infrastructure/Laravel/Controllers/StripeWebhookController.php`
- Synchronous signature validation
- Asynchronous processing dispatch
- Donation ID extraction logic
- Improved error handling

#### MollieWebhookController
**Location**: `/modules/Donation/Infrastructure/Laravel/Controllers/MollieWebhookController.php`
- Quick webhook validation
- Async job dispatch
- Payment ID to donation mapping
- Enhanced logging

### Key Improvements
- Response time < 200ms for webhook acknowledgment
- Reliable processing with retry mechanisms
- Comprehensive error logging and monitoring
- Idempotent webhook processing

## Performance Optimizations

### Queue-Specific Optimizations
1. **Memory Management**: Queue-specific memory limits prevent OOM errors
2. **Timeout Configuration**: Appropriate timeouts for different job types
3. **Priority Processing**: Critical jobs processed first
4. **Batch Processing**: Large exports processed in chunks
5. **Connection Management**: Persistent Redis connections

### Monitoring and Alerting
1. **Queue Wait Times**: Configurable thresholds with alerts
2. **Failed Job Monitoring**: Automatic notifications for critical failures
3. **Memory Usage Tracking**: Worker memory monitoring
4. **Progress Broadcasting**: Real-time updates via WebSockets

## Usage Examples

### Dispatching Jobs

```php
// Process payment webhook
ProcessPaymentWebhookJob::dispatch('stripe', $webhookData, $signature, $donationId);

// Send email notification
SendEmailJob::confirmation('user@example.com', 'Subject', $data);

// Export campaigns
ExportCampaignsJob::dispatch($filters, 'excel', $userId, $exportId);

// Cleanup expired data
CleanupExpiredDataJob::dispatch(['job_progress', 'export_files']);
```

### Monitoring Progress

```php
// Get job progress
$progress = JobProgress::find($jobId);
echo "Progress: {$progress->progress_percentage}%";
echo "Status: {$progress->status}";
echo "ETA: {$progress->estimated_completion_at}";
```

### Queue Management

```bash
# Start Horizon
php artisan horizon

# Monitor queues
php artisan horizon:status

# Clear failed jobs
php artisan queue:flush
```

## Configuration Requirements

### Environment Variables
```env
QUEUE_CONNECTION=redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_QUEUE_DB=7

# Queue worker configuration
QUEUE_HIGH_WORKERS=3
QUEUE_MEDIUM_WORKERS=2
QUEUE_LOW_WORKERS=1

# Monitoring
QUEUE_MONITORING_ENABLED=true
QUEUE_ALERT_EMAIL=admin@yourdomain.com
```

### Redis Requirements
- Redis 5.0+ recommended
- Minimum 2GB RAM allocation
- Persistent storage for reliability
- Network latency < 10ms

## Security Considerations

1. **Webhook Validation**: All webhooks validated before processing
2. **Job Serialization**: Secure job data serialization
3. **Permission Checks**: User-based job access control
4. **Data Privacy**: Configurable data inclusion in exports
5. **Audit Logging**: Comprehensive job execution logging

## Deployment Considerations

1. **Queue Workers**: Deploy with process managers (Supervisor)
2. **Redis High Availability**: Master-slave configuration
3. **Monitoring**: Set up alerts for queue failures
4. **Backup Strategy**: Include Redis data in backups
5. **Resource Scaling**: Auto-scaling based on queue depth

## Next Steps

### Pending Implementations
- GenerateReportJob for PDF/Excel report generation
- BackupDatabaseJob for automated database backups
- SendBulkNotificationJob for campaign updates
- SendWeeklyDigestJob for employee summaries
- SendReminderJob for campaign deadline notifications
- Job status dashboard for admin monitoring
- Queue management commands for worker supervision

### Future Enhancements
- Machine learning-based job prioritization
- Advanced analytics and reporting
- Multi-tenant queue isolation
- Cross-region job distribution
- Enhanced security features

This implementation provides a robust, scalable background job system capable of handling the platform's current and future needs while maintaining excellent performance and reliability.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved