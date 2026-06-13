# Usage

A comprehensive examples of how other plugins can integrate with WP Member Manager to get member IDs. I'll create integration code for common plugin types and a general integration guide.

## 1. WP MEMBER MANAGER - Add Integration API Class

First, add this file to make integration easier for other plugins:

### includes/class-integration-api.php

```php
<?php
/**
 * WP Member Manager - Integration API
 * Provides easy-to-use functions for other plugins
 */

if (!defined('ABSPATH')) {
    exit;
}

class WPMM_Integration_API {
    
    private static $instance = null;
    
    /**
     * Get singleton instance
     */
    public static function get_instance() {
        if (null === self::$instance) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    /**
     * Get current member ID (for use by other plugins)
     */
    public function get_current_member_id() {
        return apply_filters('wpmm_get_member_id', null);
    }
    
    /**
     * Get current member data
     */
    public function get_current_member() {
        return apply_filters('wpmm_get_current_member', null);
    }
    
    /**
     * Check if member is logged in
     */
    public function is_member_logged_in() {
        return apply_filters('wpmm_is_logged_in', false);
    }
    
    /**
     * Get member ID by WordPress user ID
     */
    public function get_member_id_by_wp_user($wp_user_id) {
        return get_user_meta($wp_user_id, '_wpmm_member_id', true);
    }
    
    /**
     * Get WordPress user ID by member ID
     */
    public function get_wp_user_by_member_id($member_id) {
        $users = get_users(array(
            'meta_key' => '_wpmm_member_id',
            'meta_value' => $member_id,
            'number' => 1
        ));
        
        return !empty($users) ? $users[0]->ID : null;
    }
    
    /**
     * Get member ID by email
     */
    public function get_member_id_by_email($email) {
        global $wpdb;
        
        return $wpdb->get_var($wpdb->prepare(
            "SELECT member_id FROM " . WPMM_MEMBERS_TABLE . " 
             WHERE email = %s AND status = 'active'",
            $email
        ));
    }
    
    /**
     * Check membership level
     */
    public function has_membership_level($required_level, $member_id = null) {
        if (!$member_id) {
            $member_id = $this->get_current_member_id();
        }
        
        if (!$member_id) {
            return false;
        }
        
        $levels = array('free' => 0, 'basic' => 1, 'premium' => 2, 'vip' => 3, 'admin' => 4);
        
        global $wpdb;
        $member_level = $wpdb->get_var($wpdb->prepare(
            "SELECT membership_level FROM " . WPMM_MEMBERS_TABLE . " WHERE member_id = %s",
            $member_id
        ));
        
        return ($levels[$member_level] ?? 0) >= ($levels[$required_level] ?? 0);
    }
    
    /**
     * Link a member to a WordPress user
     */
    public function link_member_to_wp_user($member_id, $wp_user_id) {
        update_user_meta($wp_user_id, '_wpmm_member_id', $member_id);
        do_action('wpmm_member_linked_to_wp_user', $member_id, $wp_user_id);
        return true;
    }
    
    /**
     * Sync member to WordPress user (creates WP user if needed)
     */
    public function sync_member_to_wp_user($member_id) {
        global $wpdb;
        
        $member = $wpdb->get_row($wpdb->prepare(
            "SELECT * FROM " . WPMM_MEMBERS_TABLE . " WHERE member_id = %s",
            $member_id
        ), ARRAY_A);
        
        if (!$member) {
            return false;
        }
        
        // Check if WP user already linked
        $wp_user_id = $this->get_wp_user_by_member_id($member_id);
        
        if ($wp_user_id) {
            return $wp_user_id;
        }
        
        // Create new WP user
        $wp_user_id = wp_insert_user(array(
            'user_login' => $member['username'],
            'user_email' => $member['email'],
            'user_pass' => wp_generate_password(),
            'first_name' => $member['first_name'],
            'last_name' => $member['last_name'],
            'display_name' => $member['display_name'],
            'role' => 'subscriber'
        ));
        
        if (!is_wp_error($wp_user_id)) {
            $this->link_member_to_wp_user($member_id, $wp_user_id);
        }
        
        return $wp_user_id;
    }
    
    /**
     * Get all members (for plugin integration)
     */
    public function get_members($args = array()) {
        $defaults = array(
            'status' => 'active',
            'membership_level' => '',
            'limit' => 50,
            'offset' => 0
        );
        
        $args = wp_parse_args($args, $defaults);
        
        global $wpdb;
        
        $where = array("status = '" . esc_sql($args['status']) . "'");
        
        if (!empty($args['membership_level'])) {
            $where[] = "membership_level = '" . esc_sql($args['membership_level']) . "'";
        }
        
        $where_clause = implode(' AND ', $where);
        
        return $wpdb->get_results(
            $wpdb->prepare(
                "SELECT * FROM " . WPMM_MEMBERS_TABLE . " 
                 WHERE {$where_clause} 
                 LIMIT %d OFFSET %d",
                $args['limit'],
                $args['offset']
            ),
            ARRAY_A
        );
    }
}

// Initialize the integration API
function wpmm_integration_api() {
    return WPMM_Integration_API::get_instance();
}
```

## 2. PAYMENT PLUGIN INTEGRATION EXAMPLE

### payment-plugin-integration.php

```php
<?php
/**
 * Payment Plugin Integration with WP Member Manager
 * 
 * This example shows how a payment plugin would use WP Member Manager
 * to store payments against member IDs
 */

class Payment_Plugin_Integration {
    
    private $table_name;
    
    public function __construct() {
        global $wpdb;
        $this->table_name = $wpdb->prefix . 'payment_transactions';
        
        // Hook into WP Member Manager actions
        add_action('init', array($this, 'initialize'));
    }
    
    /**
     * Initialize the payment integration
     */
    public function initialize() {
        // Check if WP Member Manager is active
        if (!class_exists('WPMM_Integration_API')) {
            add_action('admin_notices', function() {
                echo '<div class="notice notice-error"><p>';
                echo 'Payment Plugin requires WP Member Manager to be installed and activated.';
                echo '</p></div>';
            });
            return;
        }
    }
    
    /**
     * Process a payment
     * This is the MAIN function that uses member_id from WP Member Manager
     */
    public function process_payment($payment_data) {
        // METHOD 1: Get member ID from current logged-in member
        $member_id = wpmm_integration_api()->get_current_member_id();
        
        // METHOD 2: Get member ID by email (if provided)
        if (!$member_id && isset($payment_data['email'])) {
            $member_id = wpmm_integration_api()->get_member_id_by_email($payment_data['email']);
        }
        
        // METHOD 3: Get member ID from WordPress user
        if (!$member_id && is_user_logged_in()) {
            $member_id = wpmm_integration_api()->get_member_id_by_wp_user(get_current_user_id());
        }
        
        // Check if we have a member ID
        if (!$member_id) {
            return array(
                'success' => false,
                'message' => 'Member account required for payment'
            );
        }
        
        // Check membership level for discounts
        if (wpmm_integration_api()->has_membership_level('premium', $member_id)) {
            $payment_data['amount'] *= 0.9; // 10% discount for premium members
        }
        
        // Process payment using member_id as foreign key
        $transaction_id = $this->save_transaction($member_id, $payment_data);
        
        // Update member meta with payment info
        do_action('wpmm_update_member_meta', $member_id, 'last_payment_date', current_time('mysql'));
        do_action('wpmm_update_member_meta', $member_id, 'total_spent', 
            $this->get_total_spent($member_id) + $payment_data['amount']
        );
        
        return array(
            'success' => true,
            'transaction_id' => $transaction_id,
            'member_id' => $member_id
        );
    }
    
    /**
     * Save transaction to database
     */
    private function save_transaction($member_id, $data) {
        global $wpdb;
        
        $transaction_data = array(
            'user_id' => $member_id, // Uses WP Member Manager member_id as user_id
            'amount' => $data['amount'],
            'currency' => $data['currency'] ?? 'USD',
            'status' => 'completed',
            'payment_method' => $data['method'],
            'description' => $data['description'],
            'created_at' => current_time('mysql')
        );
        
        $wpdb->insert($this->table_name, $transaction_data);
        
        return $wpdb->insert_id;
    }
    
    /**
     * Get all transactions for a member
     */
    public function get_member_transactions($member_id = null) {
        if (!$member_id) {
            $member_id = wpmm_integration_api()->get_current_member_id();
        }
        
        if (!$member_id) {
            return array();
        }
        
        global $wpdb;
        
        return $wpdb->get_results($wpdb->prepare(
            "SELECT * FROM {$this->table_name} 
             WHERE user_id = %s 
             ORDER BY created_at DESC",
            $member_id
        ), ARRAY_A);
    }
    
    /**
     * Get total amount spent by member
     */
    public function get_total_spent($member_id) {
        global $wpdb;
        
        return $wpdb->get_var($wpdb->prepare(
            "SELECT SUM(amount) FROM {$this->table_name} 
             WHERE user_id = %s AND status = 'completed'",
            $member_id
        )) ?? 0;
    }
    
    /**
     * Create database table
     */
    public static function create_table() {
        global $wpdb;
        
        $table_name = $wpdb->prefix . 'payment_transactions';
        $charset_collate = $wpdb->get_charset_collate();
        
        $sql = "CREATE TABLE IF NOT EXISTS $table_name (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            user_id varchar(32) NOT NULL COMMENT 'Member ID from WP Member Manager',
            amount decimal(10,2) NOT NULL,
            currency varchar(3) DEFAULT 'USD',
            status varchar(20) NOT NULL DEFAULT 'pending',
            payment_method varchar(50),
            description text,
            created_at datetime DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id),
            KEY idx_user_id (user_id),
            KEY idx_status (status),
            KEY idx_created_at (created_at)
        ) $charset_collate;";
        
        require_once(ABSPATH . 'wp-admin/includes/upgrade.php');
        dbDelta($sql);
    }
}

// Usage example in payment form
function process_payment_form() {
    $payment_plugin = new Payment_Plugin_Integration();
    
    // Get current member ID
    $member_id = wpmm_integration_api()->get_current_member_id();
    
    if (!$member_id) {
        echo '<p>Please login to make a payment.</p>';
        echo do_shortcode('[wpmm_login_form]');
        return;
    }
    
    // Check if premium member for special pricing
    $is_premium = wpmm_integration_api()->has_membership_level('premium');
    
    ?>
    <form method="post" id="payment-form">
        <input type="hidden" name="member_id" value="<?php echo esc_attr($member_id); ?>">
        
        <h3>Payment Details</h3>
        <p>Member ID: <code><?php echo esc_html($member_id); ?></code></p>
        
        <?php if ($is_premium): ?>
        <p class="premium-notice">
            🎉 Premium Member Discount: 10% off!
        </p>
        <?php endif; ?>
        
        <label>Amount:</label>
        <input type="number" name="amount" required>
        
        <label>Payment Method:</label>
        <select name="payment_method">
            <option value="credit_card">Credit Card</option>
            <option value="paypal">PayPal</option>
        </select>
        
        <button type="submit">Process Payment</button>
    </form>
    <?php
}
```

## 3. LMS (LEARNING MANAGEMENT SYSTEM) INTEGRATION

### lms-plugin-integration.php

```php
<?php
/**
 * LMS Plugin Integration with WP Member Manager
 * 
 * Example of how an LMS would use member_id for course tracking
 */

class LMS_Plugin_Integration {
    
    private $enrollments_table;
    private $progress_table;
    
    public function __construct() {
        global $wpdb;
        $this->enrollments_table = $wpdb->prefix . 'lms_enrollments';
        $this->progress_table = $wpdb->prefix . 'lms_course_progress';
        
        add_action('init', array($this, 'init'));
        add_shortcode('lms_course_access', array($this, 'course_access_shortcode'));
    }
    
    public function init() {
        // Verify WP Member Manager is active
        if (!function_exists('wpmm_integration_api')) {
            return;
        }
    }
    
    /**
     * Enroll member in a course
     */
    public function enroll_member_in_course($course_id) {
        // Get member ID from WP Member Manager
        $member_id = wpmm_integration_api()->get_current_member_id();
        
        if (!$member_id) {
            return array(
                'success' => false,
                'message' => 'You must be logged in to enroll'
            );
        }
        
        // Check if already enrolled
        if ($this->is_enrolled($member_id, $course_id)) {
            return array(
                'success' => false,
                'message' => 'Already enrolled in this course'
            );
        }
        
        // Check membership level requirements
        $course_required_level = get_post_meta($course_id, '_lms_required_level', true);
        
        if ($course_required_level) {
            if (!wpmm_integration_api()->has_membership_level($course_required_level, $member_id)) {
                return array(
                    'success' => false,
                    'message' => "This course requires {$course_required_level} membership"
                );
            }
        }
        
        // Enroll using member_id as user_id
        global $wpdb;
        
        $result = $wpdb->insert(
            $this->enrollments_table,
            array(
                'user_id' => $member_id, // WP Member Manager member_id
                'course_id' => $course_id,
                'enrollment_date' => current_time('mysql'),
                'status' => 'active'
            )
        );
        
        if ($result) {
            // Hook for other systems
            do_action('lms_course_enrolled', $member_id, $course_id);
            
            return array(
                'success' => true,
                'message' => 'Successfully enrolled in course'
            );
        }
        
        return array(
            'success' => false,
            'message' => 'Enrollment failed'
        );
    }
    
    /**
     * Check if member is enrolled in course
     */
    private function is_enrolled($member_id, $course_id) {
        global $wpdb;
        
        return $wpdb->get_var($wpdb->prepare(
            "SELECT COUNT(*) FROM {$this->enrollments_table} 
             WHERE user_id = %s AND course_id = %d AND status = 'active'",
            $member_id,
            $course_id
        )) > 0;
    }
    
    /**
     * Track course progress
     */
    public function update_course_progress($course_id, $lesson_id, $progress_percentage) {
        $member_id = wpmm_integration_api()->get_current_member_id();
        
        if (!$member_id) {
            return false;
        }
        
        global $wpdb;
        
        // Check if progress record exists
        $existing = $wpdb->get_var($wpdb->prepare(
            "SELECT id FROM {$this->progress_table} 
             WHERE user_id = %s AND course_id = %d AND lesson_id = %d",
            $member_id, $course_id, $lesson_id
        ));
        
        $data = array(
            'user_id' => $member_id, // Using WPMM member_id
            'course_id' => $course_id,
            'lesson_id' => $lesson_id,
            'progress_percentage' => $progress_percentage,
            'last_accessed' => current_time('mysql')
        );
        
        if ($existing) {
            $wpdb->update(
                $this->progress_table,
                $data,
                array('id' => $existing)
            );
        } else {
            $wpdb->insert($this->progress_table, $data);
        }
        
        // Check if course completed
        if ($progress_percentage >= 100) {
            do_action('lms_lesson_completed', $member_id, $course_id, $lesson_id);
        }
        
        return true;
    }
    
    /**
     * Get member's course progress
     */
    public function get_member_course_progress($course_id) {
        $member_id = wpmm_integration_api()->get_current_member_id();
        
        if (!$member_id) {
            return array();
        }
        
        global $wpdb;
        
        return $wpdb->get_results($wpdb->prepare(
            "SELECT * FROM {$this->progress_table} 
             WHERE user_id = %s AND course_id = %d",
            $member_id, $course_id
        ), ARRAY_A);
    }
    
    /**
     * Get all enrolled courses for member
     */
    public function get_member_courses() {
        $member_id = wpmm_integration_api()->get_current_member_id();
        
        if (!$member_id) {
            return array();
        }
        
        global $wpdb;
        
        return $wpdb->get_results($wpdb->prepare(
            "SELECT e.*, p.post_title as course_name 
             FROM {$this->enrollments_table} e
             JOIN {$wpdb->posts} p ON e.course_id = p.ID
             WHERE e.user_id = %s AND e.status = 'active'",
            $member_id
        ), ARRAY_A);
    }
    
    /**
     * Course access shortcode
     */
    public function course_access_shortcode($atts, $content = null) {
        $atts = shortcode_atts(array(
            'course_id' => 0
        ), $atts);
        
        $member_id = wpmm_integration_api()->get_current_member_id();
        
        if (!$member_id) {
            return '<p>Please login to access course content.</p>';
        }
        
        if ($this->is_enrolled($member_id, $atts['course_id'])) {
            return do_shortcode($content);
        }
        
        return '<p>You are not enrolled in this course. <a href="#">Enroll Now</a></p>';
    }
    
    /**
     * Create required database tables
     */
    public static function create_tables() {
        global $wpdb;
        $charset_collate = $wpdb->get_charset_collate();
        
        // Enrollments table
        $sql_enrollments = "CREATE TABLE IF NOT EXISTS {$wpdb->prefix}lms_enrollments (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            user_id varchar(32) NOT NULL COMMENT 'Member ID from WPMM',
            course_id bigint(20) NOT NULL,
            enrollment_date datetime DEFAULT CURRENT_TIMESTAMP,
            completion_date datetime DEFAULT NULL,
            status varchar(20) DEFAULT 'active',
            PRIMARY KEY (id),
            UNIQUE KEY unique_enrollment (user_id, course_id),
            KEY idx_user_id (user_id),
            KEY idx_course_id (course_id)
        ) $charset_collate;";
        
        // Progress table
        $sql_progress = "CREATE TABLE IF NOT EXISTS {$wpdb->prefix}lms_course_progress (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            user_id varchar(32) NOT NULL COMMENT 'Member ID from WPMM',
            course_id bigint(20) NOT NULL,
            lesson_id bigint(20) NOT NULL,
            progress_percentage decimal(5,2) DEFAULT 0,
            last_accessed datetime DEFAULT CURRENT_TIMESTAMP,
            completed tinyint(1) DEFAULT 0,
            PRIMARY KEY (id),
            UNIQUE KEY unique_progress (user_id, course_id, lesson_id),
            KEY idx_user_course (user_id, course_id)
        ) $charset_collate;";
        
        require_once(ABSPATH . 'wp-admin/includes/upgrade.php');
        dbDelta($sql_enrollments);
        dbDelta($sql_progress);
    }
}

// Usage Example: Display member's courses
function display_member_courses_shortcode() {
    $lms = new LMS_Plugin_Integration();
    
    $member_id = wpmm_integration_api()->get_current_member_id();
    
    if (!$member_id) {
        return '<p>Please login to view your courses.</p>';
    }
    
    $courses = $lms->get_member_courses();
    
    ob_start();
    ?>
    <div class="member-courses">
        <h3>Your Courses (Member ID: <?php echo esc_html($member_id); ?>)</h3>
        
        <?php if (empty($courses)): ?>
            <p>You are not enrolled in any courses yet.</p>
        <?php else: ?>
            <ul class="course-list">
                <?php foreach ($courses as $course): ?>
                <li>
                    <strong><?php echo esc_html($course['course_name']); ?></strong>
                    <br>Enrolled: <?php echo date_i18n(get_option('date_format'), strtotime($course['enrollment_date'])); ?>
                    <br>Status: <?php echo esc_html($course['status']); ?>
                </li>
                <?php endforeach; ?>
            </ul>
        <?php endif; ?>
    </div>
    <?php
    return ob_get_clean();
}
add_shortcode('my_courses', 'display_member_courses_shortcode');
```

## 4. MEMBERSHIP/SUBSCRIPTION PLUGIN INTEGRATION

### subscription-plugin-integration.php

```php
<?php
/**
 * Subscription Plugin Integration with WP Member Manager
 * 
 * Example showing how to use member_id for subscription management
 */

class Subscription_Plugin_Integration {
    
    private $subscriptions_table;
    
    public function __construct() {
        global $wpdb;
        $this->subscriptions_table = $wpdb->prefix . 'member_subscriptions';
        
        // Hook into member events
        add_action('wpmm_member_created', array($this, 'create_free_subscription'));
        add_action('wpmm_member_deleted', array($this, 'cancel_all_subscriptions'));
    }
    
    /**
     * Create subscription for member
     */
    public function create_subscription($plan_id, $duration_days = 30) {
        $member_id = wpmm_integration_api()->get_current_member_id();
        
        if (!$member_id) {
            return array('success' => false, 'message' => 'Login required');
        }
        
        // Check if already has active subscription
        $current_sub = $this->get_active_subscription($member_id);
        if ($current_sub) {
            return array('success' => false, 'message' => 'Active subscription exists');
        }
        
        global $wpdb;
        
        $subscription_data = array(
            'user_id' => $member_id, // WP Member Manager member_id
            'plan_id' => $plan_id,
            'start_date' => current_time('mysql'),
            'end_date' => date('Y-m-d H:i:s', strtotime("+{$duration_days} days")),
            'status' => 'active',
            'auto_renew' => 1
        );
        
        $wpdb->insert($this->subscriptions_table, $subscription_data);
        $subscription_id = $wpdb->insert_id;
        
        // Update member membership level
        $plan = $this->get_plan($plan_id);
        do_action('wpmm_update_member', $member_id, array(
            'membership_level' => $plan['level']
        ));
        
        do_action('subscription_created', $member_id, $subscription_id);
        
        return array(
            'success' => true,
            'subscription_id' => $subscription_id,
            'member_id' => $member_id
        );
    }
    
    /**
     * Get active subscription for member
     */
    public function get_active_subscription($member_id = null) {
        if (!$member_id) {
            $member_id = wpmm_integration_api()->get_current_member_id();
        }
        
        if (!$member_id) {
            return null;
        }
        
        global $wpdb;
        
        return $wpdb->get_row($wpdb->prepare(
            "SELECT * FROM {$this->subscriptions_table} 
             WHERE user_id = %s 
             AND status = 'active' 
             AND end_date > %s
             ORDER BY id DESC LIMIT 1",
            $member_id,
            current_time('mysql')
        ), ARRAY_A);
    }
    
    /**
     * Cancel subscription
     */
    public function cancel_subscription($member_id = null) {
        if (!$member_id) {
            $member_id = wpmm_integration_api()->get_current_member_id();
        }
        
        if (!$member_id) {
            return false;
        }
        
        global $wpdb;
        
        $wpdb->update(
            $this->subscriptions_table,
            array(
                'status' => 'cancelled',
                'cancelled_date' => current_time('mysql')
            ),
            array(
                'user_id' => $member_id,
                'status' => 'active'
            )
        );
        
        // Revert to free membership
        do_action('wpmm_update_member', $member_id, array(
            'membership_level' => 'free'
        ));
        
        return true;
    }
    
    /**
     * Create free subscription on member creation
     */
    public function create_free_subscription($member_id) {
        global $wpdb;
        
        $wpdb->insert(
            $this->subscriptions_table,
            array(
                'user_id' => $member_id,
                'plan_id' => 1, // Free plan
                'start_date' => current_time('mysql'),
                'end_date' => date('Y-m-d H:i:s', strtotime('+100 years')),
                'status' => 'active',
                'auto_renew' => 0
            )
        );
    }
    
    /**
     * Cancel all subscriptions when member deleted
     */
    public function cancel_all_subscriptions($member_id) {
        global $wpdb;
        
        $wpdb->update(
            $this->subscriptions_table,
            array('status' => 'cancelled'),
            array('user_id' => $member_id, 'status' => 'active')
        );
    }
    
    /**
     * Get subscription history
     */
    public function get_subscription_history($member_id = null) {
        if (!$member_id) {
            $member_id = wpmm_integration_api()->get_current_member_id();
        }
        
        if (!$member_id) {
            return array();
        }
        
        global $wpdb;
        
        return $wpdb->get_results($wpdb->prepare(
            "SELECT * FROM {$this->subscriptions_table} 
             WHERE user_id = %s 
             ORDER BY id DESC",
            $member_id
        ), ARRAY_A);
    }
    
    /**
     * Create database tables
     */
    public static function create_tables() {
        global $wpdb;
        $charset_collate = $wpdb->get_charset_collate();
        
        $sql = "CREATE TABLE IF NOT EXISTS {$wpdb->prefix}member_subscriptions (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            user_id varchar(32) NOT NULL COMMENT 'Member ID from WPMM',
            plan_id int(11) NOT NULL,
            start_date datetime DEFAULT CURRENT_TIMESTAMP,
            end_date datetime NOT NULL,
            status varchar(20) DEFAULT 'active',
            auto_renew tinyint(1) DEFAULT 1,
            cancelled_date datetime DEFAULT NULL,
            PRIMARY KEY (id),
            KEY idx_user_id (user_id),
            KEY idx_status_end (status, end_date)
        ) $charset_collate;";
        
        require_once(ABSPATH . 'wp-admin/includes/upgrade.php');
        dbDelta($sql);
    }
    
    private function get_plan($plan_id) {
        // Placeholder - load from your plans table
        return array('level' => 'basic');
    }
}
```

## 5. UNIVERSAL INTEGRATION HELPER

### quick-integration-helper.php

```php
<?php
/**
 * Quick Integration Helper
 * 
 * This file can be included in ANY plugin to quickly get member IDs
 */

if (!function_exists('get_current_member_id')) {
    /**
     * Get current member ID from WP Member Manager
     * 
     * @return string|null Member ID or null if not logged in
     */
    function get_current_member_id() {
        // Check if WP Member Manager is active
        if (function_exists('wpmm_integration_api')) {
            return wpmm_integration_api()->get_current_member_id();
        }
        
        // Fallback: Try direct filter
        return apply_filters('wpmm_get_member_id', null);
    }
}

if (!function_exists('get_member_id_by_wp_user')) {
    /**
     * Get member ID from WordPress user ID
     * 
     * @param int $wp_user_id WordPress user ID
     * @return string|null Member ID
     */
    function get_member_id_by_wp_user($wp_user_id) {
        if (function_exists('wpmm_integration_api')) {
            return wpmm_integration_api()->get_member_id_by_wp_user($wp_user_id);
        }
        
        return get_user_meta($wp_user_id, '_wpmm_member_id', true);
    }
}

if (!function_exists('is_wpmm_member_logged_in')) {
    /**
     * Check if a member is logged in
     * 
     * @return bool
     */
    function is_wpmm_member_logged_in() {
        return get_current_member_id() !== null;
    }
}

if (!function_exists('save_with_member_id')) {
    /**
     * Save data with member ID as foreign key
     * 
     * @param string $table Database table name
     * @param array $data Data to save
     * @return int|false Insert ID or false on failure
     */
    function save_with_member_id($table, $data) {
        $member_id = get_current_member_id();
        
        if (!$member_id) {
            return false;
        }
        
        // Add member_id to data
        $data['user_id'] = $member_id;
        $data['created_at'] = current_time('mysql');
        
        global $wpdb;
        
        $result = $wpdb->insert($table, $data);
        
        if ($result) {
            return $wpdb->insert_id;
        }
        
        return false;
    }
}

if (!function_exists('query_by_member_id')) {
    /**
     * Query data by member ID
     * 
     * @param string $table Database table
     * @param string $user_id_column Column name for user_id (default: 'user_id')
     * @return array Results
     */
    function query_by_member_id($table, $user_id_column = 'user_id') {
        $member_id = get_current_member_id();
        
        if (!$member_id) {
            return array();
        }
        
        global $wpdb;
        
        return $wpdb->get_results($wpdb->prepare(
            "SELECT * FROM {$table} WHERE {$user_id_column} = %s",
            $member_id
        ), ARRAY_A);
    }
}
```

## 6. HOW TO USE IN YOUR OWN PLUGINS

### README-INTEGRATION.md

```markdown
# Integrating with WP Member Manager

## Quick Start

### 1. Check if WP Member Manager is Active

```php
if (function_exists('wpmm_integration_api')) {
    // WP Member Manager is active
    $member_id = wpmm_integration_api()->get_current_member_id();
}
```

### 2. Get Current Member ID

```php
// Method 1: Direct API call
$member_id = wpmm_integration_api()->get_current_member_id();

// Method 2: WordPress filter
$member_id = apply_filters('wpmm_get_member_id', null);

// Method 3: From WordPress user
$member_id = get_user_meta(get_current_user_id(), '_wpmm_member_id', true);
```

### 3. Store Data with Member ID

```php
global $wpdb;

$member_id = wpmm_integration_api()->get_current_member_id();

$wpdb->insert(
    $wpdb->prefix . 'my_plugin_table',
    array(
        'user_id' => $member_id, // Use member_id as user_id
        'data' => $my_data,
        'created_at' => current_time('mysql')
    )
);
```

### 4. Query Data by Member ID

```php
$member_id = wpmm_integration_api()->get_current_member_id();

$results = $wpdb->get_results($wpdb->prepare(
    "SELECT * FROM {$wpdb->prefix}my_plugin_table 
     WHERE user_id = %s",
    $member_id
));
```

## Available API Functions

| Function | Description |
|----------|-------------|
| `wpmm_integration_api()->get_current_member_id()` | Get current logged-in member's ID |
| `wpmm_integration_api()->get_current_member()` | Get full member data |
| `wpmm_integration_api()->is_member_logged_in()` | Check if member is logged in |
| `wpmm_integration_api()->has_membership_level('premium')` | Check membership level |
| `wpmm_integration_api()->get_member_id_by_email($email)` | Get member ID by email |
| `wpmm_integration_api()->get_member_id_by_wp_user($user_id)` | Get member ID from WP user |

## Hooks Available

```php
// Action when member created
do_action('wpmm_member_created', $member_id);

// Action when member updated
do_action('wpmm_member_updated', $member_id, $updated_data);

// Action when member deleted
do_action('wpmm_member_deleted', $member_id);

// Filter for member ID
apply_filters('wpmm_get_member_id', $member_id);

// Filter to check login status
apply_filters('wpmm_is_logged_in', false);
```

## Database Structure

When creating tables in your plugin, use `user_id VARCHAR(32)` to store WP Member Manager IDs:

```sql
CREATE TABLE wp_your_plugin_table (
    id bigint(20) NOT NULL AUTO_INCREMENT,
    user_id varchar(32) NOT NULL COMMENT 'WP Member Manager member_id',
    -- your other columns
    created_at datetime DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_user_id (user_id)
);
```


## Complete Working Example

### example-plugin-that-uses-member-id.php

```php
<?php
/**
 * Plugin Name: Example Plugin Using Member ID
 * Description: Demonstrates how to use WP Member Manager member_id
 * Version: 1.0.0
 */

class Example_Member_Plugin {
    
    private $table_name;

```
