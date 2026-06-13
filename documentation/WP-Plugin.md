# WP-Plugin
I'll create the complete, production-ready WordPress Member Manager plugin with all files. This is a comprehensive plugin ready for shared hosting deployment.

## 1. MAIN PLUGIN FILE - wp-member-manager.php

```php
<?php
/**
 * Plugin Name: WP Member Manager
 * Plugin URI: https://yourwebsite.com/plugins/wp-member-manager
 * Description: Comprehensive member account management system with cross-plugin user ID sharing
 * Version: 1.0.0
 * Author: Your Name
 * Author URI: https://yourwebsite.com
 * License: GPL v2 or later
 * Text Domain: wp-member-manager
 * Domain Path: /languages
 * Requires at least: 5.8
 * Requires PHP: 7.4
 */

// Prevent direct access
if (!defined('ABSPATH')) {
    exit;
}

// Define plugin constants
define('WPMM_VERSION', '1.0.0');
define('WPMM_PLUGIN_DIR', plugin_dir_path(__FILE__));
define('WPMM_PLUGIN_URL', plugin_dir_url(__FILE__));
define('WPMM_PLUGIN_BASENAME', plugin_basename(__FILE__));
define('WPMM_MINIMUM_WP_VERSION', '5.8');
define('WPMM_MINIMUM_PHP_VERSION', '7.4');

// Database table names with WordPress prefix
global $wpdb;
define('WPMM_MEMBERS_TABLE', $wpdb->prefix . 'wpmm_members');
define('WPMM_SESSIONS_TABLE', $wpdb->prefix . 'wpmm_sessions');
define('WPMM_META_TABLE', $wpdb->prefix . 'wpmm_member_meta');
define('WPMM_SECURITY_LOG_TABLE', $wpdb->prefix . 'wpmm_security_log');

/**
 * Main plugin initialization class
 */
final class WP_Member_Manager_Main {
    
    /**
     * Single instance of the class
     */
    private static $instance = null;
    
    /**
     * Plugin components
     */
    private $auth;
    private $crud;
    private $session;
    private $validator;
    private $hooks;
    private $notifications;
    private $id_provider;
    
    /**
     * Get single instance
     */
    public static function get_instance() {
        if (null === self::$instance) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    /**
     * Constructor
     */
    private function __construct() {
        $this->check_requirements();
        $this->load_dependencies();
        $this->init_components();
        $this->setup_hooks();
        
        // Load text domain
        add_action('init', array($this, 'load_textdomain'));
    }
    
    /**
     * Check plugin requirements
     */
    private function check_requirements() {
        // Check WordPress version
        if (version_compare(get_bloginfo('version'), WPMM_MINIMUM_WP_VERSION, '<')) {
            add_action('admin_notices', function() {
                echo '<div class="notice notice-error"><p>';
                printf(
                    __('WP Member Manager requires WordPress version %s or higher. Please upgrade WordPress.', 'wp-member-manager'),
                    WPMM_MINIMUM_WP_VERSION
                );
                echo '</p></div>';
            });
            
            // Deactivate plugin
            deactivate_plugins(WPMM_PLUGIN_BASENAME);
            return;
        }
        
        // Check PHP version
        if (version_compare(PHP_VERSION, WPMM_MINIMUM_PHP_VERSION, '<')) {
            add_action('admin_notices', function() {
                echo '<div class="notice notice-error"><p>';
                printf(
                    __('WP Member Manager requires PHP version %s or higher. Please upgrade PHP.', 'wp-member-manager'),
                    WPMM_MINIMUM_PHP_VERSION
                );
                echo '</p></div>';
            });
            
            deactivate_plugins(WPMM_PLUGIN_BASENAME);
            return;
        }
    }
    
    /**
     * Load required files
     */
    private function load_dependencies() {
        $includes_dir = WPMM_PLUGIN_DIR . 'includes/';
        
        // Core classes
        require_once $includes_dir . 'class-member-crud.php';
        require_once $includes_dir . 'class-member-validator.php';
        require_once $includes_dir . 'class-member-session.php';
        require_once $includes_dir . 'class-member-auth.php';
        require_once $includes_dir . 'class-member-notifications.php';
        require_once $includes_dir . 'class-member-id-provider.php';
        require_once $includes_dir . 'class-member-hooks.php';
        
        // Template loader
        require_once $includes_dir . 'class-template-loader.php';
        
        // Admin interface
        if (is_admin()) {
            require_once $includes_dir . 'class-admin-interface.php';
        }
    }
    
    /**
     * Initialize plugin components
     */
    private function init_components() {
        $this->crud = new WP_Member_CRUD();
        $this->validator = new WP_Member_Validator();
        $this->session = new WP_Member_Session($this->crud);
        $this->auth = new WP_Member_Auth($this->crud, $this->session);
        $this->notifications = new WP_Member_Notifications();
        $this->id_provider = new WP_Member_ID_Provider($this->crud);
        $this->hooks = new WP_Member_Hooks($this);
    }
    
    /**
     * Setup WordPress hooks
     */
    private function setup_hooks() {
        // Activation and deactivation
        register_activation_hook(__FILE__, array($this, 'activate'));
        register_deactivation_hook(__FILE__, array($this, 'deactivate'));
        
        // Enqueue scripts and styles
        add_action('wp_enqueue_scripts', array($this, 'enqueue_scripts'));
        add_action('admin_enqueue_scripts', array($this, 'enqueue_admin_scripts'));
        
        // AJAX handlers
        add_action('wp_ajax_nopriv_wpmm_login', array($this, 'ajax_login'));
        add_action('wp_ajax_nopriv_wpmm_register', array($this, 'ajax_register'));
        add_action('wp_ajax_nopriv_wpmm_reset_password', array($this, 'ajax_reset_password'));
        add_action('wp_ajax_wpmm_update_profile', array($this, 'ajax_update_profile'));
        add_action('wp_ajax_wpmm_logout', array($this, 'ajax_logout'));
        
        // Schedule cron jobs
        add_action('wp', array($this, 'schedule_cron_jobs'));
        add_action('wpmm_hourly_cleanup', array($this, 'hourly_cleanup'));
        add_action('wpmm_daily_maintenance', array($this, 'daily_maintenance'));
        
        // Session validation
        add_action('init', array($this->session, 'validate_current_session'), 1);
    }
    
    /**
     * Plugin activation
     */
    public function activate() {
        $this->create_database_tables();
        $this->set_default_options();
        $this->create_default_pages();
        
        // Flush rewrite rules
        flush_rewrite_rules();
        
        // Set activation flag for redirect
        add_option('wpmm_activation_redirect', true);
    }
    
    /**
     * Plugin deactivation
     */
    public function deactivate() {
        // Clear scheduled events
        wp_clear_scheduled_hook('wpmm_hourly_cleanup');
        wp_clear_scheduled_hook('wpmm_daily_maintenance');
        
        // Clean up
        delete_option('wpmm_activation_redirect');
        
        // Flush rewrite rules
        flush_rewrite_rules();
    }
    
    /**
     * Create database tables
     */
    private function create_database_tables() {
        global $wpdb;
        
        $charset_collate = $wpdb->get_charset_collate();
        
        // Members table
        $sql_members = "CREATE TABLE IF NOT EXISTS " . WPMM_MEMBERS_TABLE . " (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            member_id varchar(32) NOT NULL UNIQUE,
            username varchar(60) NOT NULL UNIQUE,
            email varchar(100) NOT NULL UNIQUE,
            password_hash varchar(255) NOT NULL,
            first_name varchar(50) DEFAULT '',
            last_name varchar(50) DEFAULT '',
            display_name varchar(100) DEFAULT '',
            membership_level varchar(20) DEFAULT 'free',
            status enum('active','pending','suspended','deleted') DEFAULT 'pending',
            email_verified tinyint(1) DEFAULT 0,
            registration_date datetime DEFAULT CURRENT_TIMESTAMP,
            last_login datetime DEFAULT NULL,
            last_activity datetime DEFAULT NULL,
            login_attempts int(11) DEFAULT 0,
            locked_until datetime DEFAULT NULL,
            deleted_at datetime DEFAULT NULL,
            PRIMARY KEY (id),
            KEY idx_member_id (member_id),
            KEY idx_username (username),
            KEY idx_email (email),
            KEY idx_status (status),
            KEY idx_membership (membership_level)
        ) $charset_collate;";
        
        // Sessions table
        $sql_sessions = "CREATE TABLE IF NOT EXISTS " . WPMM_SESSIONS_TABLE . " (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            session_id varchar(128) NOT NULL UNIQUE,
            member_id varchar(32) NOT NULL,
            ip_address varchar(45) DEFAULT '',
            user_agent text DEFAULT '',
            login_time datetime DEFAULT CURRENT_TIMESTAMP,
            last_activity datetime DEFAULT CURRENT_TIMESTAMP,
            expiry_time datetime NOT NULL,
            is_active tinyint(1) DEFAULT 1,
            PRIMARY KEY (id),
            KEY idx_session_id (session_id),
            KEY idx_member_active (member_id, is_active),
            KEY idx_expiry (expiry_time)
        ) $charset_collate;";
        
        // Member meta table
        $sql_meta = "CREATE TABLE IF NOT EXISTS " . WPMM_META_TABLE . " (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            member_id varchar(32) NOT NULL,
            meta_key varchar(255) NOT NULL,
            meta_value longtext DEFAULT '',
            PRIMARY KEY (id),
            UNIQUE KEY unique_member_meta (member_id, meta_key),
            KEY idx_member_meta (member_id, meta_key)
        ) $charset_collate;";
        
        // Security log table
        $sql_security = "CREATE TABLE IF NOT EXISTS " . WPMM_SECURITY_LOG_TABLE . " (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            member_id varchar(32) DEFAULT '',
            event_type varchar(50) NOT NULL,
            ip_address varchar(45) DEFAULT '',
            details text DEFAULT '',
            created_at datetime DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id),
            KEY idx_member (member_id),
            KEY idx_event (event_type),
            KEY idx_created (created_at)
        ) $charset_collate;";
        
        require_once(ABSPATH . 'wp-admin/includes/upgrade.php');
        dbDelta($sql_members);
        dbDelta($sql_sessions);
        dbDelta($sql_meta);
        dbDelta($sql_security);
        
        // Store database version
        update_option('wpmm_db_version', WPMM_VERSION);
    }
    
    /**
     * Set default options
     */
    private function set_default_options() {
        $default_options = array(
            'wpmm_session_duration' => 86400, // 24 hours
            'wpmm_max_login_attempts' => 5,
            'wpmm_lockout_duration' => 900, // 15 minutes
            'wpmm_require_email_verification' => 'yes',
            'wpmm_default_membership_level' => 'free',
            'wpmm_enable_recaptcha' => 'no',
            'wpmm_recaptcha_site_key' => '',
            'wpmm_recaptcha_secret_key' => '',
            'wpmm_welcome_email_subject' => 'Welcome to {site_name}',
            'wpmm_welcome_email_message' => 'Hello {first_name},\n\nWelcome to {site_name}! Your account has been created successfully.\n\nPlease click the link below to verify your email:\n{verification_link}\n\nThank you!',
            'wpmm_password_reset_subject' => 'Password Reset - {site_name}',
            'wpmm_password_reset_message' => 'Hello {first_name},\n\nYou requested a password reset. Click the link below:\n{reset_link}\n\nIf you did not request this, please ignore this email.',
            'wpmm_pages_login' => 0,
            'wpmm_pages_register' => 0,
            'wpmm_pages_dashboard' => 0,
            'wpmm_pages_profile' => 0,
        );
        
        foreach ($default_options as $option_name => $option_value) {
            if (get_option($option_name) === false) {
                update_option($option_name, $option_value);
            }
        }
    }
    
    /**
     * Create default pages
     */
    private function create_default_pages() {
        $pages = array(
            'member-login' => array(
                'title' => 'Member Login',
                'content' => '[wpmm_login_form]',
                'option' => 'wpmm_pages_login'
            ),
            'member-register' => array(
                'title' => 'Member Registration',
                'content' => '[wpmm_register_form]',
                'option' => 'wpmm_pages_register'
            ),
            'member-dashboard' => array(
                'title' => 'Member Dashboard',
                'content' => '[wpmm_dashboard]',
                'option' => 'wpmm_pages_dashboard'
            ),
            'member-profile' => array(
                'title' => 'Edit Profile',
                'content' => '[wpmm_profile_edit]',
                'option' => 'wpmm_pages_profile'
            ),
        );
        
        foreach ($pages as $slug => $page_data) {
            // Check if page exists
            $existing_page = get_page_by_path($slug);
            
            if (!$existing_page) {
                $page_id = wp_insert_post(array(
                    'post_title' => $page_data['title'],
                    'post_content' => $page_data['content'],
                    'post_status' => 'publish',
                    'post_type' => 'page',
                    'post_name' => $slug,
                    'comment_status' => 'closed',
                    'ping_status' => 'closed',
                ));
                
                if ($page_id && !is_wp_error($page_id)) {
                    update_option($page_data['option'], $page_id);
                }
            } else {
                update_option($page_data['option'], $existing_page->ID);
            }
        }
    }
    
    /**
     * Load plugin textdomain
     */
    public function load_textdomain() {
        load_plugin_textdomain(
            'wp-member-manager',
            false,
            dirname(WPMM_PLUGIN_BASENAME) . '/languages'
        );
    }
    
    /**
     * Enqueue frontend scripts and styles
     */
    public function enqueue_scripts() {
        // CSS
        wp_enqueue_style(
            'wpmm-frontend',
            WPMM_PLUGIN_URL . 'assets/css/frontend.css',
            array(),
            WPMM_VERSION
        );
        
        // JavaScript
        wp_enqueue_script(
            'wpmm-frontend',
            WPMM_PLUGIN_URL . 'assets/js/frontend.js',
            array('jquery'),
            WPMM_VERSION,
            true
        );
        
        // Localize script
        wp_localize_script('wpmm-frontend', 'wpmm_ajax', array(
            'ajax_url' => admin_url('admin-ajax.php'),
            'nonce' => wp_create_nonce('wpmm_frontend_nonce'),
            'login_url' => $this->get_page_url('login'),
            'register_url' => $this->get_page_url('register'),
            'dashboard_url' => $this->get_page_url('dashboard'),
            'profile_url' => $this->get_page_url('profile'),
            'is_logged_in' => $this->session->is_logged_in(),
            'current_member_id' => $this->session->get_current_member_id(),
            'messages' => array(
                'login_success' => __('Login successful! Redirecting...', 'wp-member-manager'),
                'register_success' => __('Registration successful! Please check your email.', 'wp-member-manager'),
                'logout_success' => __('Logged out successfully!', 'wp-member-manager'),
                'error_general' => __('An error occurred. Please try again.', 'wp-member-manager'),
                'password_mismatch' => __('Passwords do not match.', 'wp-member-manager'),
                'password_strength' => __('Password must be at least 8 characters with uppercase, lowercase, number, and special character.', 'wp-member-manager'),
                'required_field' => __('This field is required.', 'wp-member-manager'),
                'invalid_email' => __('Please enter a valid email address.', 'wp-member-manager'),
            )
        ));
    }
    
    /**
     * Enqueue admin scripts and styles
     */
    public function enqueue_admin_scripts($hook) {
        if (strpos($hook, 'wp-member-manager') !== false) {
            wp_enqueue_style(
                'wpmm-admin',
                WPMM_PLUGIN_URL . 'assets/css/admin.css',
                array(),
                WPMM_VERSION
            );
            
            wp_enqueue_script(
                'wpmm-admin',
                WPMM_PLUGIN_URL . 'assets/js/admin.js',
                array('jquery'),
                WPMM_VERSION,
                true
            );
        }
    }
    
    /**
     * Schedule cron jobs
     */
    public function schedule_cron_jobs() {
        if (!wp_next_scheduled('wpmm_hourly_cleanup')) {
            wp_schedule_event(time(), 'hourly', 'wpmm_hourly_cleanup');
        }
        
        if (!wp_next_scheduled('wpmm_daily_maintenance')) {
            wp_schedule_event(time(), 'daily', 'wpmm_daily_maintenance');
        }
    }
    
    /**
     * Hourly cleanup tasks
     */
    public function hourly_cleanup() {
        global $wpdb;
        
        // Clean expired sessions
        $wpdb->query($wpdb->prepare(
            "UPDATE " . WPMM_SESSIONS_TABLE . " 
             SET is_active = 0 
             WHERE expiry_time < %s",
            current_time('mysql')
        ));
        
        // Reset login attempts for expired lockouts
        $wpdb->query($wpdb->prepare(
            "UPDATE " . WPMM_MEMBERS_TABLE . " 
             SET login_attempts = 0, locked_until = NULL 
             WHERE locked_until IS NOT NULL 
             AND locked_until < %s",
            current_time('mysql')
        ));
        
        // Clear old transients
        $this->cleanup_transients();
    }
    
    /**
     * Daily maintenance tasks
     */
    public function daily_maintenance() {
        global $wpdb;
        
        // Archive old security logs (keep 90 days)
        $cutoff_date = date('Y-m-d H:i:s', strtotime('-90 days'));
        $wpdb->query($wpdb->prepare(
            "DELETE FROM " . WPMM_SECURITY_LOG_TABLE . " 
             WHERE created_at < %s",
            $cutoff_date
        ));
        
        // Delete inactive sessions older than 7 days
        $old_sessions = date('Y-m-d H:i:s', strtotime('-7 days'));
        $wpdb->query($wpdb->prepare(
            "DELETE FROM " . WPMM_SESSIONS_TABLE . " 
             WHERE is_active = 0 
             AND last_activity < %s",
            $old_sessions
        ));
        
        // Update member statistics
        $this->update_member_statistics();
    }
    
    /**
     * Clean up expired transients
     */
    private function cleanup_transients() {
        global $wpdb;
        
        $time_threshold = time();
        $wpdb->query(
            "DELETE FROM {$wpdb->options} 
             WHERE option_name LIKE '\_transient\_timeout\_wpmm\_%' 
             AND option_value < {$time_threshold}"
        );
    }
    
    /**
     * Update member statistics
     */
    private function update_member_statistics() {
        global $wpdb;
        
        // Count members by status
        $stats = $wpdb->get_results(
            "SELECT status, COUNT(*) as count 
             FROM " . WPMM_MEMBERS_TABLE . " 
             GROUP BY status"
        );
        
        // Count members by level
        $levels = $wpdb->get_results(
            "SELECT membership_level, COUNT(*) as count 
             FROM " . WPMM_MEMBERS_TABLE . " 
             WHERE status = 'active' 
             GROUP BY membership_level"
        );
        
        // Store statistics
        $statistics = array(
            'total_members' => 0,
            'active_members' => 0,
            'pending_members' => 0,
            'by_level' => array(),
            'last_updated' => current_time('timestamp')
        );
        
        foreach ($stats as $stat) {
            $statistics[$stat->status . '_members'] = $stat->count;
            $statistics['total_members'] += $stat->count;
        }
        
        foreach ($levels as $level) {
            $statistics['by_level'][$level->membership_level] = $level->count;
        }
        
        update_option('wpmm_member_statistics', $statistics);
    }
    
    /**
     * AJAX login handler
     */
    public function ajax_login() {
        // Verify nonce
        if (!wp_verify_nonce($_POST['nonce'], 'wpmm_frontend_nonce')) {
            wp_send_json_error(array('message' => 'Security check failed'));
        }
        
        $username = sanitize_text_field($_POST['username']);
        $password = $_POST['password'];
        $remember = isset($_POST['remember']);
        
        $result = $this->auth->login($username, $password, $remember);
        
        if ($result['success']) {
            wp_send_json_success($result);
        } else {
            wp_send_json_error($result);
        }
    }
    
    /**
     * AJAX registration handler
     */
    public function ajax_register() {
        // Verify nonce
        if (!wp_verify_nonce($_POST['nonce'], 'wpmm_frontend_nonce')) {
            wp_send_json_error(array('message' => 'Security check failed'));
        }
        
        // Validate input
        $validation = $this->validator->validate_registration($_POST);
        
        if (!$validation['valid']) {
            wp_send_json_error(array(
                'message' => 'Validation failed',
                'errors' => $validation['errors']
            ));
        }
        
        // Create member
        $result = $this->crud->create_member($validation['data']);
        
        if ($result['success']) {
            // Send verification email if required
            if (get_option('wpmm_require_email_verification') === 'yes') {
                $this->notifications->send_verification_email($result['member_id']);
            }
            
            wp_send_json_success(array(
                'message' => 'Registration successful',
                'member_id' => $result['member_id']
            ));
        } else {
            wp_send_json_error(array(
                'message' => $result['message']
            ));
        }
    }
    
    /**
     * AJAX password reset handler
     */
    public function ajax_reset_password() {
        if (!wp_verify_nonce($_POST['nonce'], 'wpmm_frontend_nonce')) {
            wp_send_json_error(array('message' => 'Security check failed'));
        }
        
        $email = sanitize_email($_POST['email']);
        
        $result = $this->auth->initiate_password_reset($email);
        
        if ($result['success']) {
            wp_send_json_success(array(
                'message' => 'If the email exists, a password reset link has been sent.'
            ));
        } else {
            wp_send_json_error(array('message' => $result['message']));
        }
    }
    
    /**
     * AJAX profile update handler
     */
    public function ajax_update_profile() {
        if (!wp_verify_nonce($_POST['nonce'], 'wpmm_frontend_nonce')) {
            wp_send_json_error(array('message' => 'Security check failed'));
        }
        
        // Check if logged in
        $member_id = $this->session->get_current_member_id();
        if (!$member_id) {
            wp_send_json_error(array('message' => 'You must be logged in'));
        }
        
        // Validate and update
        $result = $this->crud->update_member($member_id, $_POST);
        
        if ($result['success']) {
            wp_send_json_success(array('message' => 'Profile updated successfully'));
        } else {
            wp_send_json_error(array('message' => $result['message']));
        }
    }
    
    /**
     * AJAX logout handler
     */
    public function ajax_logout() {
        $this->session->destroy_current_session();
        wp_send_json_success(array('message' => 'Logged out successfully'));
    }
    
    /**
     * Get page URL by type
     */
    public function get_page_url($type) {
        $page_id = 0;
        
        switch ($type) {
            case 'login':
                $page_id = get_option('wpmm_pages_login');
                break;
            case 'register':
                $page_id = get_option('wpmm_pages_register');
                break;
            case 'dashboard':
                $page_id = get_option('wpmm_pages_dashboard');
                break;
            case 'profile':
                $page_id = get_option('wpmm_pages_profile');
                break;
        }
        
        if ($page_id) {
            return get_permalink($page_id);
        }
        
        return home_url('/');
    }
    
    /**
     * Get plugin component
     */
    public function get_component($name) {
        if (property_exists($this, $name)) {
            return $this->$name;
        }
        return null;
    }
    
    /**
     * Prevent cloning
     */
    private function __clone() {}
    
    /**
     * Prevent unserialization
     */
    public function __wakeup() {
        throw new \Exception('Cannot unserialize singleton');
    }
}

/**
 * Initialize the plugin
 */
function wpmm_init() {
    return WP_Member_Manager_Main::get_instance();
}

// Start the plugin
$GLOBALS['wp_member_manager'] = wpmm_init();
```

## 2. DATABASE SCHEMA - database/schema.sql

```sql
-- WP Member Manager Database Schema
-- Version 1.0.0

-- Members Table
CREATE TABLE IF NOT EXISTS `wp_wpmm_members` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `member_id` varchar(32) NOT NULL,
    `username` varchar(60) NOT NULL,
    `email` varchar(100) NOT NULL,
    `password_hash` varchar(255) NOT NULL,
    `first_name` varchar(50) DEFAULT '',
    `last_name` varchar(50) DEFAULT '',
    `display_name` varchar(100) DEFAULT '',
    `membership_level` varchar(20) DEFAULT 'free',
    `status` enum('active','pending','suspended','deleted') DEFAULT 'pending',
    `email_verified` tinyint(1) DEFAULT 0,
    `registration_date` datetime DEFAULT CURRENT_TIMESTAMP,
    `last_login` datetime DEFAULT NULL,
    `last_activity` datetime DEFAULT NULL,
    `login_attempts` int(11) DEFAULT 0,
    `locked_until` datetime DEFAULT NULL,
    `deleted_at` datetime DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_member_id` (`member_id`),
    UNIQUE KEY `uk_username` (`username`),
    UNIQUE KEY `uk_email` (`email`),
    KEY `idx_status` (`status`),
    KEY `idx_membership` (`membership_level`),
    KEY `idx_registration_date` (`registration_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Sessions Table
CREATE TABLE IF NOT EXISTS `wp_wpmm_sessions` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `session_id` varchar(128) NOT NULL,
    `member_id` varchar(32) NOT NULL,
    `ip_address` varchar(45) DEFAULT '',
    `user_agent` text DEFAULT '',
    `login_time` datetime DEFAULT CURRENT_TIMESTAMP,
    `last_activity` datetime DEFAULT CURRENT_TIMESTAMP,
    `expiry_time` datetime NOT NULL,
    `is_active` tinyint(1) DEFAULT 1,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_session_id` (`session_id`),
    KEY `idx_member_active` (`member_id`, `is_active`),
    KEY `idx_expiry` (`expiry_time`),
    KEY `idx_last_activity` (`last_activity`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Member Meta Table
CREATE TABLE IF NOT EXISTS `wp_wpmm_member_meta` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `member_id` varchar(32) NOT NULL,
    `meta_key` varchar(255) NOT NULL,
    `meta_value` longtext DEFAULT '',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_member_meta` (`member_id`, `meta_key`),
    KEY `idx_member` (`member_id`),
    KEY `idx_meta_key` (`meta_key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Security Log Table
CREATE TABLE IF NOT EXISTS `wp_wpmm_security_log` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `member_id` varchar(32) DEFAULT '',
    `event_type` varchar(50) NOT NULL,
    `ip_address` varchar(45) DEFAULT '',
    `details` text DEFAULT '',
    `created_at` datetime DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `idx_member` (`member_id`),
    KEY `idx_event_type` (`event_type`),
    KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Indexes for performance
ALTER TABLE `wp_wpmm_members` ADD INDEX `idx_email_status` (`email`, `status`);
ALTER TABLE `wp_wpmm_sessions` ADD INDEX `idx_member_expiry` (`member_id`, `expiry_time`);
ALTER TABLE `wp_wpmm_member_meta` ADD INDEX `idx_member_key_value` (`member_id`, `meta_key`, `meta_value`(100));
```

## 3. MEMBER CRUD CLASS - includes/class-member-crud.php

```php
<?php
/**
 * Member CRUD Operations
 */

if (!defined('ABSPATH')) {
    exit;
}

class WP_Member_CRUD {
    
    /**
     * WordPress database instance
     */
    private $wpdb;
    private $members_table;
    private $meta_table;
    private $sessions_table;
    
    /**
     * Constructor
     */
    public function __construct() {
        global $wpdb;
        $this->wpdb = $wpdb;
        $this->members_table = WPMM_MEMBERS_TABLE;
        $this->meta_table = WPMM_META_TABLE;
        $this->sessions_table = WPMM_SESSIONS_TABLE;
    }
    
    /**
     * Create a new member
     */
    public function create_member($data) {
        // Generate unique member ID
        $member_id = $this->generate_member_id();
        
        // Prepare member data
        $member_data = array(
            'member_id' => $member_id,
            'username' => sanitize_user($data['username']),
            'email' => sanitize_email($data['email']),
            'password_hash' => wp_hash_password($data['password']),
            'first_name' => sanitize_text_field($data['first_name']),
            'last_name' => sanitize_text_field($data['last_name']),
            'display_name' => sanitize_text_field($data['first_name'] . ' ' . $data['last_name']),
            'membership_level' => get_option('wpmm_default_membership_level', 'free'),
            'status' => get_option('wpmm_require_email_verification') === 'yes' ? 'pending' : 'active',
            'email_verified' => get_option('wpmm_require_email_verification') === 'yes' ? 0 : 1,
            'registration_date' => current_time('mysql'),
            'login_attempts' => 0
        );
        
        // Insert member
        $result = $this->wpdb->insert(
            $this->members_table,
            $member_data,
            array('%s', '%s', '%s', '%s', '%s', '%s', '%s', '%s', '%s', '%d', '%s', '%d')
        );
        
        if ($result === false) {
            return array(
                'success' => false,
                'message' => 'Database error: ' . $this->wpdb->last_error
            );
        }
        
        // Save additional meta data
        $meta_data = array(
            'registration_ip' => $this->get_client_ip(),
            'registration_user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'timezone' => $data['timezone'] ?? wp_timezone_string(),
        );
        
        foreach ($meta_data as $key => $value) {
            $this->update_member_meta($member_id, $key, $value);
        }
        
        // Generate verification token if needed
        if (get_option('wpmm_require_email_verification') === 'yes') {
            $token = $this->generate_verification_token($member_id);
            $this->update_member_meta($member_id, 'verification_token', $token);
            $this->update_member_meta($member_id, 'verification_token_expiry', 
                date('Y-m-d H:i:s', strtotime('+48 hours'))
            );
        }
        
        do_action('wpmm_member_created', $member_id, $member_data);
        
        return array(
            'success' => true,
            'member_id' => $member_id,
            'message' => 'Member created successfully'
        );
    }
    
    /**
     * Get member by ID
     */
    public function get_member($member_id) {
        $member = $this->wpdb->get_row(
            $this->wpdb->prepare(
                "SELECT * FROM {$this->members_table} WHERE member_id = %s",
                $member_id
            ),
            ARRAY_A
        );
        
        if ($member) {
            // Load meta data
            $member['meta'] = $this->get_member_meta($member_id);
        }
        
        return $member;
    }
    
    /**
     * Get member by username or email
     */
    public function get_member_by_login($login) {
        $member = $this->wpdb->get_row(
            $this->wpdb->prepare(
                "SELECT * FROM {$this->members_table} 
                 WHERE (username = %s OR email = %s) 
                 AND status != 'deleted' 
                 LIMIT 1",
                $login, $login
            ),
            ARRAY_A
        );
        
        if ($member) {
            $member['meta'] = $this->get_member_meta($member['member_id']);
        }
        
        return $member;
    }
    
    /**
     * Update member data
     */
    public function update_member($member_id, $data) {
        $allowed_fields = array(
            'first_name', 'last_name', 'display_name', 'email',
            'membership_level', 'status'
        );
        
        $update_data = array();
        $format = array();
        
        foreach ($allowed_fields as $field) {
            if (isset($data[$field])) {
                $update_data[$field] = sanitize_text_field($data[$field]);
                $format[] = '%s';
            }
        }
        
        if (empty($update_data)) {
            return array(
                'success' => false,
                'message' => 'No valid fields to update'
            );
        }
        
        // Add last activity timestamp
        $update_data['last_activity'] = current_time('mysql');
        $format[] = '%s';
        
        $result = $this->wpdb->update(
            $this->members_table,
            $update_data,
            array('member_id' => $member_id),
            $format,
            array('%s')
        );
        
        if ($result === false) {
            return array(
                'success' => false,
                'message' => 'Update failed: ' . $this->wpdb->last_error
            );
        }
        
        do_action('wpmm_member_updated', $member_id, $update_data);
        
        return array(
            'success' => true,
            'message' => 'Member updated successfully'
        );
    }
    
    /**
     * Update member password
     */
    public function update_password($member_id, $new_password) {
        $result = $this->wpdb->update(
            $this->members_table,
            array(
                'password_hash' => wp_hash_password($new_password),
                'last_activity' => current_time('mysql')
            ),
            array('member_id' => $member_id),
            array('%s', '%s'),
            array('%s')
        );
        
        if ($result === false) {
            return array(
                'success' => false,
                'message' => 'Password update failed'
            );
        }
        
        // Invalidate all existing sessions
        $this->invalidate_member_sessions($member_id);
        
        return array(
            'success' => true,
            'message' => 'Password updated successfully'
        );
    }
    
    /**
     * Delete member (soft delete)
     */
    public function delete_member($member_id) {
        // Soft delete - update status
        $result = $this->wpdb->update(
            $this->members_table,
            array(
                'status' => 'deleted',
                'deleted_at' => current_time('mysql')
            ),
            array('member_id' => $member_id),
            array('%s', '%s'),
            array('%s')
        );
        
        if ($result === false) {
            return array(
                'success' => false,
                'message' => 'Delete failed'
            );
        }
        
        // Invalidate sessions
        $this->invalidate_member_sessions($member_id);
        
        do_action('wpmm_member_deleted', $member_id);
        
        return array(
            'success' => true,
            'message' => 'Member deleted successfully'
        );
    }
    
    /**
     * Permanently delete member and all related data
     */
    public function permanently_delete_member($member_id) {
        $this->wpdb->delete($this->meta_table, array('member_id' => $member_id), array('%s'));
        $this->wpdb->delete($this->sessions_table, array('member_id' => $member_id), array('%s'));
        $this->wpdb->delete($this->members_table, array('member_id' => $member_id), array('%s'));
        
        return array(
            'success' => true,
            'message' => 'Member permanently deleted'
        );
    }
    
    /**
     * Get all members with pagination
     */
    public function get_members($args = array()) {
        $defaults = array(
            'status' => 'active',
            'membership_level' => '',
            'search' => '',
            'orderby' => 'registration_date',
            'order' => 'DESC',
            'limit' => 20,
            'offset' => 0
        );
        
        $args = wp_parse_args($args, $defaults);
        
        $where = array('1=1');
        
        if (!empty($args['status'])) {
            $where[] = $this->wpdb->prepare('status = %s', $args['status']);
        }
        
        if (!empty($args['membership_level'])) {
            $where[] = $this->wpdb->prepare('membership_level = %s', $args['membership_level']);
        }
        
        if (!empty($args['search'])) {
            $search = '%' . $this->wpdb->esc_like($args['search']) . '%';
            $where[] = $this->wpdb->prepare(
                '(username LIKE %s OR email LIKE %s OR display_name LIKE %s)',
                $search, $search, $search
            );
        }
        
        $where_clause = implode(' AND ', $where);
        
        // Get total count
        $total = $this->wpdb->get_var(
            "SELECT COUNT(*) FROM {$this->members_table} WHERE {$where_clause}"
        );
        
        // Get results
        $members = $this->wpdb->get_results(
            $this->wpdb->prepare(
                "SELECT * FROM {$this->members_table} 
                 WHERE {$where_clause} 
                 ORDER BY {$args['orderby']} {$args['order']} 
                 LIMIT %d OFFSET %d",
                $args['limit'], $args['offset']
            ),
            ARRAY_A
        );
        
        return array(
            'members' => $members,
            'total' => $total,
            'limit' => $args['limit'],
            'offset' => $args['offset']
        );
    }
    
    /**
     * Check if username exists
     */
    public function username_exists($username) {
        $count = $this->wpdb->get_var(
            $this->wpdb->prepare(
                "SELECT COUNT(*) FROM {$this->members_table} 
                 WHERE username = %s AND status != 'deleted'",
                $username
            )
        );
        
        return $count > 0;
    }
    
    /**
     * Check if email exists
     */
    public function email_exists($email) {
        $count = $this->wpdb->get_var(
            $this->wpdb->prepare(
                "SELECT COUNT(*) FROM {$this->members_table} 
                 WHERE email = %s AND status != 'deleted'",
                $email
            )
        );
        
        return $count > 0;
    }
    
    /**
     * Get member meta
     */
    public function get_member_meta($member_id, $key = '') {
        if (!empty($key)) {
            $value = $this->wpdb->get_var(
                $this->wpdb->prepare(
                    "SELECT meta_value FROM {$this->meta_table} 
                     WHERE member_id = %s AND meta_key = %s",
                    $member_id, $key
                )
            );
            
            return $value ? maybe_unserialize($value) : null;
        }
        
        // Get all meta for member
        $meta = $this->wpdb->get_results(
            $this->wpdb->prepare(
                "SELECT meta_key, meta_value FROM {$this->meta_table} 
                 WHERE member_id = %s",
                $member_id
            ),
            ARRAY_A
        );
        
        $formatted_meta = array();
        foreach ($meta as $item) {
            $formatted_meta[$item['meta_key']] = maybe_unserialize($item['meta_value']);
        }
        
        return $formatted_meta;
    }
    
    /**
     * Update member meta
     */
    public function update_member_meta($member_id, $key, $value) {
        $exists = $this->wpdb->get_var(
            $this->wpdb->prepare(
                "SELECT id FROM {$this->meta_table} 
                 WHERE member_id = %s AND meta_key = %s",
                $member_id, $key
            )
        );
        
        $value = maybe_serialize($value);
        
        if ($exists) {
            $result = $this->wpdb->update(
                $this->meta_table,
                array('meta_value' => $value),
                array('member_id' => $member_id, 'meta_key' => $key),
                array('%s'),
                array('%s', '%s')
            );
        } else {
            $result = $this->wpdb->insert(
                $this->meta_table,
                array(
                    'member_id' => $member_id,
                    'meta_key' => $key,
                    'meta_value' => $value
                ),
                array('%s', '%s', '%s')
            );
        }
        
        return $result !== false;
    }
    
    /**
     * Delete member meta
     */
    public function delete_member_meta($member_id, $key) {
        return $this->wpdb->delete(
            $this->meta_table,
            array('member_id' => $member_id, 'meta_key' => $key),
            array('%s', '%s')
        );
    }
    
    /**
     * Record login attempt
     */
    public function record_login_attempt($member_id, $success) {
        $member = $this->get_member($member_id);
        
        if (!$member) {
            return;
        }
        
        if ($success) {
            // Reset on successful login
            $this->wpdb->update(
                $this->members_table,
                array(
                    'login_attempts' => 0,
                    'locked_until' => null,
                    'last_login' => current_time('mysql'),
                    'last_activity' => current_time('mysql')
                ),
                array('member_id' => $member_id),
                array('%d', null, '%s', '%s'),
                array('%s')
            );
        } else {
            // Increment on failed login
            $attempts = $member['login_attempts'] + 1;
            $max_attempts = get_option('wpmm_max_login_attempts', 5);
            
            $update_data = array(
                'login_attempts' => $attempts,
                'last_activity' => current_time('mysql')
            );
            
            $format = array('%d', '%s');
            
            // Lock account if max attempts reached
            if ($attempts >= $max_attempts) {
                $lockout_duration = get_option('wpmm_lockout_duration', 900);
                $locked_until = date('Y-m-d H:i:s', time() + $lockout_duration);
                $update_data['locked_until'] = $locked_until;
                $format[] = '%s';
            }
            
            $this->wpdb->update(
                $this->members_table,
                $update_data,
                array('member_id' => $member_id),
                $format,
                array('%s')
            );
        }
    }
    
    /**
     * Verify member email
     */
    public function verify_email($member_id, $token) {
        $stored_token = $this->get_member_meta($member_id, 'verification_token');
        $token_expiry = $this->get_member_meta($member_id, 'verification_token_expiry');
        
        if (!$stored_token || $stored_token !== $token) {
            return array(
                'success' => false,
                'message' => 'Invalid verification token'
            );
        }
        
        if (strtotime($token_expiry) < time()) {
            return array(
                'success' => false,
                'message' => 'Verification token has expired'
            );
        }
        
        // Update member status
        $result = $this->wpdb->update(
            $this->members_table,
            array(
                'status' => 'active',
                'email_verified' => 1,
                'last_activity' => current_time('mysql')
            ),
            array('member_id' => $member_id),
            array('%s', '%d', '%s'),
            array('%s')
        );
        
        if ($result === false) {
            return array(
                'success' => false,
                'message' => 'Failed to verify email'
            );
        }
        
        // Clean up tokens
        $this->delete_member_meta($member_id, 'verification_token');
        $this->delete_member_meta($member_id, 'verification_token_expiry');
        
        do_action('wpmm_member_verified', $member_id);
        
        return array(
            'success' => true,
            'message' => 'Email verified successfully'
        );
    }
    
    /**
     * Generate unique member ID
     */
    private function generate_member_id() {
        $member_id = '';
        $exists = true;
        
        while ($exists) {
            $member_id = 'mm_' . wp_generate_password(26, false);
            $exists = $this->wpdb->get_var(
                $this->wpdb->prepare(
                    "SELECT COUNT(*) FROM {$this->members_table} WHERE member_id = %s",
                    $member_id
                )
            );
        }
        
        return $member_id;
    }
    
    /**
     * Generate verification token
     */
    private function generate_verification_token($member_id) {
        return wp_hash($member_id . time() . wp_rand(), 'nonce');
    }
    
    /**
     * Get client IP address
     */
    private function get_client_ip() {
        $ip = '';
        
        if (!empty($_SERVER['HTTP_CLIENT_IP'])) {
            $ip = $_SERVER['HTTP_CLIENT_IP'];
        } elseif (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
            $ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
        } else {
            $ip = $_SERVER['REMOTE_ADDR'] ?? '';
        }
        
        return sanitize_text_field($ip);
    }
    
    /**
     * Invalidate all sessions for a member
     */
    private function invalidate_member_sessions($member_id) {
        return $this->wpdb->update(
            $this->sessions_table,
            array('is_active' => 0),
            array('member_id' => $member_id, 'is_active' => 1),
            array('%d'),
            array('%s', '%d')
        );
    }
    
    /**
     * Get member count by status
     */
    public function get_member_count($status = 'active') {
        return $this->wpdb->get_var(
            $this->wpdb->prepare(
                "SELECT COUNT(*) FROM {$this->members_table} WHERE status = %s",
                $status
            )
        );
    }
}
```

## 4. MEMBER SESSION CLASS - includes/class-member-session.php

```php
<?php
/**
 * Member Session Management
 */

if (!defined('ABSPATH')) {
    exit;
}

class WP_Member_Session {
    
    private $wpdb;
    private $sessions_table;
    private $members_table;
    private $crud;
    private $current_member = null;
    private $current_session = null;
    
    /**
     * Constructor
     */
    public function __construct($crud) {
        global $wpdb;
        $this->wpdb = $wpdb;
        $this->sessions_table = WPMM_SESSIONS_TABLE;
        $this->members_table = WPMM_MEMBERS_TABLE;
        $this->crud = $crud;
    }
    
    /**
     * Create new session
     */
    public function create_session($member_id, $remember = false) {
        $session_id = $this->generate_session_id();
        $session_duration = get_option('wpmm_session_duration', 86400);
        
        // Extend session for "remember me"
        if ($remember) {
            $session_duration = 30 * DAY_IN_SECONDS;
        }
        
        $expiry_time = date('Y-m-d H:i:s', time() + $session_duration);
        
        // Limit concurrent sessions
        $this->limit_concurrent_sessions($member_id);
        
        // Insert session record
        $result = $this->wpdb->insert(
            $this->sessions_table,
            array(
                'session_id' => $session_id,
                'member_id' => $member_id,
                'ip_address' => $this->get_client_ip(),
                'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
                'login_time' => current_time('mysql'),
                'last_activity' => current_time('mysql'),
                'expiry_time' => $expiry_time,
                'is_active' => 1
            ),
            array('%s', '%s', '%s', '%s', '%s', '%s', '%s', '%d')
        );
        
        if ($result === false) {
            return array(
                'success' => false,
                'message' => 'Failed to create session'
            );
        }
        
        // Set session cookie
        $this->set_session_cookie($session_id, $expiry_time);
        
        // Store session in property
        $this->current_session = array(
            'session_id' => $session_id,
            'member_id' => $member_id,
            'expiry_time' => $expiry_time
        );
        
        return array(
            'success' => true,
            'session_id' => $session_id,
            'expiry_time' => $expiry_time
        );
    }
    
    /**
     * Validate current session from cookie
     */
    public function validate_current_session() {
        // Skip if already validated
        if ($this->current_member !== null) {
            return;
        }
        
        // Get session ID from cookie
        $session_id = $_COOKIE['wpmm_session'] ?? '';
        
        if (empty($session_id)) {
            return;
        }
        
        $session = $this->get_session($session_id);
        
        if (!$session) {
            $this->destroy_session_cookie();
            return;
        }
        
        // Check if session is active
        if (!$session['is_active']) {
            $this->destroy_session_cookie();
            return;
        }
        
        // Check if session has expired
        if (strtotime($session['expiry_time']) < time()) {
            $this->deactivate_session($session_id);
            $this->destroy_session_cookie();
            return;
        }
        
        // Check IP consistency (basic security)
        if ($this->is_ip_suspicious($session['ip_address'])) {
            $this->log_security_event('suspicious_ip', $session['member_id'], 
                "IP changed from {$session['ip_address']} to " . $this->get_client_ip()
            );
        }
        
        // Get member data
        $member = $this->crud->get_member($session['member_id']);
        
        if (!$member || $member['status'] !== 'active') {
            $this->deactivate_session($session_id);
            $this->destroy_session_cookie();
            return;
        }
        
        // Update last activity
        $this->update_session_activity($session_id);
        
        // Extend session if close to expiry
        $this->extend_session_if_needed($session_id, $session['expiry_time']);
        
        $this->current_session = $session;
        $this->current_member = $member;
    }
    
    /**
     * Get session by ID
     */
    public function get_session($session_id) {
        return $this->wpdb->get_row(
            $this->wpdb->prepare(
                "SELECT * FROM {$this->sessions_table} WHERE session_id = %s",
                $session_id
            ),
            ARRAY_A
        );
    }
    
    /**
     * Get current member
     */
    public function get_current_member() {
        return $this->current_member;
    }
    
    /**
     * Get current member ID
     */
    public function get_current_member_id() {
        return $this->current_member['member_id'] ?? null;
    }
    
    /**
     * Check if member is logged in
     */
    public function is_logged_in() {
        return $this->current_member !== null;
    }
    
    /**
     * Require member login
     */
    public function require_login() {
        if (!$this->is_logged_in()) {
            // Store intended URL
            $current_url = (isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? "https" : "http") . 
                          "://$_SERVER[HTTP_HOST]$_SERVER[REQUEST_URI]";
            
            setcookie('wpmm_redirect', $current_url, time() + 900, COOKIEPATH, COOKIE_DOMAIN);
            
            // Redirect to login page
            $login_page = get_option('wpmm_pages_login');
            if ($login_page) {
                wp_redirect(get_permalink($login_page));
            } else {
                wp_redirect(wp_login_url($current_url));
            }
            exit;
        }
    }
    
    /**
     * Destroy current session
     */
    public function destroy_current_session() {
        $session_id = $_COOKIE['wpmm_session'] ?? '';
        
        if (!empty($session_id)) {
            $this->deactivate_session($session_id);
        }
        
        $this->destroy_session_cookie();
        $this->current_member = null;
        $this->current_session = null;
    }
    
    /**
     * Deactivate a session
     */
    private function deactivate_session($session_id) {
        return $this->wpdb->update(
            $this->sessions_table,
            array('is_active' => 0),
            array('session_id' => $session_id),
            array('%d'),
            array('%s')
        );
    }
    
    /**
     * Update session activity timestamp
     */
    private function update_session_activity($session_id) {
        $this->wpdb->update(
            $this->sessions_table,
            array('last_activity' => current_time('mysql')),
            array('session_id' => $session_id),
            array('%s'),
            array('%s')
        );
    }
    
    /**
     * Extend session if close to expiry
     */
    private function extend_session_if_needed($session_id, $expiry_time) {
        $remaining = strtotime($expiry_time) - time();
        
        // Extend if less than 1 hour remaining
        if ($remaining < HOUR_IN_SECONDS && $remaining > 0) {
            $new_expiry = date('Y-m-d H:i:s', time() + get_option('wpmm_session_duration', 86400));
            
            $this->wpdb->update(
                $this->sessions_table,
                array('expiry_time' => $new_expiry),
                array('session_id' => $session_id),
                array('%s'),
                array('%s')
            );
            
            // Update cookie
            $this->set_session_cookie($session_id, $new_expiry);
        }
    }
    
    /**
     * Limit concurrent sessions per member
     */
    private function limit_concurrent_sessions($member_id) {
        $max_sessions = 5;
        
        $active_sessions = $this->wpdb->get_results(
            $this->wpdb->prepare(
                "SELECT id, login_time FROM {$this->sessions_table} 
                 WHERE member_id = %s AND is_active = 1 
                 ORDER BY login_time ASC",
                $member_id
            ),
            ARRAY_A
        );
        
        if (count($active_sessions) >= $max_sessions) {
            // Deactivate oldest sessions
            $sessions_to_remove = array_slice($active_sessions, 0, count($active_sessions) - $max_sessions + 1);
            
            foreach ($sessions_to_remove as $session) {
                $this->wpdb->update(
                    $this->sessions_table,
                    array('is_active' => 0),
                    array('id' => $session['id']),
                    array('%d'),
                    array('%d')
                );
            }
        }
    }
    
    /**
     * Set session cookie
     */
    private function set_session_cookie($session_id, $expiry_time) {
        $expiry_timestamp = is_numeric($expiry_time) ? $expiry_time : strtotime($expiry_time);
        
        setcookie(
            'wpmm_session',
            $session_id,
            array(
                'expires' => $expiry_timestamp,
                'path' => COOKIEPATH,
                'domain' => COOKIE_DOMAIN,
                'secure' => is_ssl(),
                'httponly' => true,
                'samesite' => 'Lax'
            )
        );
    }
    
    /**
     * Destroy session cookie
     */
    private function destroy_session_cookie() {
        setcookie(
            'wpmm_session',
            '',
            array(
                'expires' => time() - 3600,
                'path' => COOKIEPATH,
                'domain' => COOKIE_DOMAIN,
                'secure' => is_ssl(),
                'httponly' => true,
                'samesite' => 'Lax'
            )
        );
        
        // Remove redirect cookie if exists
        if (isset($_COOKIE['wpmm_redirect'])) {
            setcookie('wpmm_redirect', '', time() - 3600, COOKIEPATH, COOKIE_DOMAIN);
        }
    }
    
    /**
     * Check if IP change is suspicious
     */
    private function is_ip_suspicious($session_ip) {
        $current_ip = $this->get_client_ip();
        return $session_ip !== $current_ip;
    }
    
    /**
     * Get client IP address
     */
    private function get_client_ip() {
        $ip = '';
        
        if (!empty($_SERVER['HTTP_CLIENT_IP'])) {
            $ip = $_SERVER['HTTP_CLIENT_IP'];
        } elseif (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
            $ip = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR'])[0];
        } else {
            $ip = $_SERVER['REMOTE_ADDR'] ?? '';
        }
        
        return sanitize_text_field(trim($ip));
    }
    
    /**
     * Generate secure session ID
     */
    private function generate_session_id() {
        return bin2hex(random_bytes(32));
    }
    
    /**
     * Log security event
     */
    private function log_security_event($event_type, $member_id, $details = '') {
        global $wpdb;
        
        $wpdb->insert(
            WPMM_SECURITY_LOG_TABLE,
            array(
                'member_id' => $member_id,
                'event_type' => $event_type,
                'ip_address' => $this->get_client_ip(),
                'details' => $details,
                'created_at' => current_time('mysql')
            ),
            array('%s', '%s', '%s', '%s', '%s')
        );
    }
}
```

## 5. TEMPLATE LOADER - includes/class-template-loader.php

```php
<?php
/**
 * Template Loader for Member Manager
 */

if (!defined('ABSPATH')) {
    exit;
}

class WPMM_Template_Loader {
    
    /**
     * Load template file
     */
    public static function get_template($template_name, $args = array()) {
        $template_path = self::locate_template($template_name);
        
        if (!$template_path) {
            return '';
        }
        
        // Extract args for use in template
        if (!empty($args) && is_array($args)) {
            extract($args);
        }
        
        ob_start();
        include $template_path;
        return ob_get_clean();
    }
    
    /**
     * Locate template file
     */
    private static function locate_template($template_name) {
        // Check theme directory first
        $theme_template = get_stylesheet_directory() . '/wp-member-manager/' . $template_name;
        
        if (file_exists($theme_template)) {
            return $theme_template;
        }
        
        // Check parent theme directory
        $parent_theme_template = get_template_directory() . '/wp-member-manager/' . $template_name;
        
        if (file_exists($parent_theme_template)) {
            return $parent_theme_template;
        }
        
        // Fall back to plugin templates directory
        $plugin_template = WPMM_PLUGIN_DIR . 'templates/' . $template_name;
        
        if (file_exists($plugin_template)) {
            return $plugin_template;
        }
        
        return false;
    }
}
```

## 6. TEMPLATE - LOGIN FORM - templates/login-form.php

```php
<?php
/**
 * Member Login Form Template
 */

if (!defined('ABSPATH')) {
    exit;
}

$redirect = $_GET['redirect_to'] ?? '';
if (empty($redirect) && isset($_COOKIE['wpmm_redirect'])) {
    $redirect = $_COOKIE['wpmm_redirect'];
}
?>

<div class="wpmm-container">
    <div class="wpmm-form-wrapper">
        <h2><?php _e('Member Login', 'wp-member-manager'); ?></h2>
        
        <div class="wpmm-message" style="display: none;"></div>
        
        <form id="wpmm-login-form" class="wpmm-form" method="post">
            <?php wp_nonce_field('wpmm_login', 'wpmm_login_nonce'); ?>
            <input type="hidden" name="redirect" value="<?php echo esc_attr($redirect); ?>">
            
            <div class="wpmm-form-group">
                <label for="wpmm-username">
                    <?php _e('Username or Email', 'wp-member-manager'); ?>
                </label>
                <input 
                    type="text" 
                    id="wpmm-username" 
                    name="username" 
                    class="wpmm-input" 
                    required 
                    autocomplete="username"
                >
            </div>
            
            <div class="wpmm-form-group">
                <label for="wpmm-password">
                    <?php _e('Password', 'wp-member-manager'); ?>
                </label>
                <input 
                    type="password" 
                    id="wpmm-password" 
                    name="password" 
                    class="wpmm-input" 
                    required 
                    autocomplete="current-password"
                >
            </div>
            
            <div class="wpmm-form-group wpmm-remember-me">
                <label>
                    <input type="checkbox" name="remember" value="1">
                    <?php _e('Remember Me', 'wp-member-manager'); ?>
                </label>
            </div>
            
            <div class="wpmm-form-group">
                <button type="submit" class="wpmm-button wpmm-button-primary">
                    <?php _e('Login', 'wp-member-manager'); ?>
                </button>
            </div>
            
            <div class="wpmm-form-links">
                <a href="<?php echo esc_url(wp_lostpassword_url()); ?>" class="wpmm-link">
                    <?php _e('Forgot Password?', 'wp-member-manager'); ?>
                </a>
                
                <?php
                $register_page = get_option('wpmm_pages_register');
                if ($register_page): ?>
                    <a href="<?php echo get_permalink($register_page); ?>" class="wpmm-link">
                        <?php _e('Create Account', 'wp-member-manager'); ?>
                    </a>
                <?php endif; ?>
            </div>
        </form>
    </div>
</div>
```

## 7. TEMPLATE - REGISTER FORM - templates/register-form.php

```php
<?php
/**
 * Member Registration Form Template
 */

if (!defined('ABSPATH')) {
    exit;
}
?>

<div class="wpmm-container">
    <div class="wpmm-form-wrapper">
        <h2><?php _e('Create Member Account', 'wp-member-manager'); ?></h2>
        
        <div class="wpmm-message" style="display: none;"></div>
        
        <form id="wpmm-register-form" class="wpmm-form" method="post">
            <?php wp_nonce_field('wpmm_register', 'wpmm_register_nonce'); ?>
            
            <div class="wpmm-form-row">
                <div class="wpmm-form-group wpmm-col-half">
                    <label for="wpmm-first-name">
                        <?php _e('First Name', 'wp-member-manager'); ?> *
                    </label>
                    <input 
                        type="text" 
                        id="wpmm-first-name" 
                        name="first_name" 
                        class="wpmm-input" 
                        required
                    >
                </div>
                
                <div class="wpmm-form-group wpmm-col-half">
                    <label for="wpmm-last-name">
                        <?php _e('Last Name', 'wp-member-manager'); ?> *
                    </label>
                    <input 
                        type="text" 
                        id="wpmm-last-name" 
                        name="last_name" 
                        class="wpmm-input" 
                        required
                    >
                </div>
            </div>
            
            <div class="wpmm-form-group">
                <label for="wpmm-reg-username">
                    <?php _e('Username', 'wp-member-manager'); ?> *
                </label>
                <input 
                    type="text" 
                    id="wpmm-reg-username" 
                    name="username" 
                    class="wpmm-input" 
                    required 
                    minlength="3" 
                    maxlength="50"
                >
                <span class="wpmm-field-message">
                    <?php _e('Only letters, numbers, and underscores', 'wp-member-manager'); ?>
                </span>
            </div>
            
            <div class="wpmm-form-group">
                <label for="wpmm-reg-email">
                    <?php _e('Email Address', 'wp-member-manager'); ?> *
                </label>
                <input 
                    type="email" 
                    id="wpmm-reg-email" 
                    name="email" 
                    class="wpmm-input" 
                    required
                >
            </div>
            
            <div class="wpmm-form-row">
                <div class="wpmm-form-group wpmm-col-half">
                    <label for="wpmm-reg-password">
                        <?php _e('Password', 'wp-member-manager'); ?> *
                    </label>
                    <input 
                        type="password" 
                        id="wpmm-reg-password" 
                        name="password" 
                        class="wpmm-input" 
                        required 
                        minlength="8"
                    >
                </div>
                
                <div class="wpmm-form-group wpmm-col-half">
                    <label for="wpmm-reg-confirm-password">
                        <?php _e('Confirm Password', 'wp-member-manager'); ?> *
                    </label>
                    <input 
                        type="password" 
                        id="wpmm-reg-confirm-password" 
                        name="confirm_password" 
                        class="wpmm-input" 
                        required
                    >
                </div>
            </div>
            
            <div class="wpmm-password-requirements">
                <p><?php _e('Password must contain:', 'wp-member-manager'); ?></p>
                <ul>
                    <li><?php _e('At least 8 characters', 'wp-member-manager'); ?></li>
                    <li><?php _e('One uppercase letter', 'wp-member-manager'); ?></li>
                    <li><?php _e('One lowercase letter', 'wp-member-manager'); ?></li>
                    <li><?php _e('One number', 'wp-member-manager'); ?></li>
                    <li><?php _e('One special character (!@#$%^&*)', 'wp-member-manager'); ?></li>
                </ul>
            </div>
            
            <div class="wpmm-form-group wpmm-gdpr-consent">
                <label>
                    <input type="checkbox" name="gdpr_consent" value="1" required>
                    <?php _e('I agree to the Terms of Service and Privacy Policy', 'wp-member-manager'); ?> *
                </label>
            </div>
            
            <!-- Honeypot for spam protection -->
            <div class="wpmm-honeypot" style="display: none;">
                <input type="text" name="website" tabindex="-1" autocomplete="off">
            </div>
            
            <div class="wpmm-form-group">
                <button type="submit" class="wpmm-button wpmm-button-primary">
                    <?php _e('Create Account', 'wp-member-manager'); ?>
                </button>
            </div>
            
            <div class="wpmm-form-links">
                <?php
                $login_page = get_option('wpmm_pages_login');
                if ($login_page): ?>
                    <p>
                        <?php _e('Already have an account?', 'wp-member-manager'); ?>
                        <a href="<?php echo get_permalink($login_page); ?>" class="wpmm-link">
                            <?php _e('Login here', 'wp-member-manager'); ?>
                        </a>
                    </p>
                <?php endif; ?>
            </div>
        </form>
    </div>
</div>
```

## 8. FRONTEND CSS - assets/css/frontend.css

```css
/**
 * WP Member Manager - Frontend Styles
 */

.wpmm-container {
    max-width: 600px;
    margin: 40px auto;
    padding: 0 15px;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen-Sans, Ubuntu, Cantarell, "Helvetica Neue", sans-serif;
}

.wpmm-form-wrapper {
    background: #ffffff;
    border-radius: 8px;
    padding: 40px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.wpmm-form-wrapper h2 {
    margin-top: 0;
    margin-bottom: 30px;
    color: #333;
    font-size: 24px;
    text-align: center;
}

/* Form Elements */
.wpmm-form-group {
    margin-bottom: 20px;
}

.wpmm-form-group label {
    display: block;
    margin-bottom: 8px;
    color: #555;
    font-weight: 500;
    font-size: 14px;
}

.wpmm-input {
    width: 100%;
    padding: 12px 15px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 16px;
    transition: border-color 0.3s ease;
    box-sizing: border-box;
}

.wpmm-input:focus {
    outline: none;
    border-color: #0073aa;
    box-shadow: 0 0 0 2px rgba(0, 115, 170, 0.1);
}

.wpmm-input.error {
    border-color: #dc3545;
}

/* Form Row */
.wpmm-form-row {
    display: flex;
    gap: 20px;
    margin-bottom: 20px;
}

.wpmm-col-half {
    flex: 1;
}

/* Buttons */
.wpmm-button {
    display: inline-block;
    padding: 12px 30px;
    border: none;
    border-radius: 4px;
    font-size: 16px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    text-decoration: none;
}

.wpmm-button-primary {
    background-color: #0073aa;
    color: #ffffff;
    width: 100%;
}

.wpmm-button-primary:hover {
    background-color: #005a87;
}

.wpmm-button-primary:disabled {
    background-color: #ccc;
    cursor: not-allowed;
}

/* Messages */
.wpmm-message {
    padding: 15px;
    margin-bottom: 20px;
    border-radius: 4px;
    font-size: 14px;
}

.wpmm-message-success {
    background-color: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
}

.wpmm-message-error {
    background-color: #f8d7da;
    color: #721c24;
    border: 1px solid #f5c6cb;
}

.wpmm-message-info {
    background-color: #d1ecf1;
    color: #0c5460;
    border: 1px solid #bee5eb;
}

/* Field Messages */
.wpmm-field-message {
    display: block;
    margin-top: 5px;
    font-size: 12px;
    color: #666;
}

.wpmm-field-error {
    display: block;
    margin-top: 5px;
    font-size: 12px;
    color: #dc3545;
}

/* Remember Me & Checkboxes */
.wpmm-remember-me label {
    display: flex;
    align-items: center;
    gap: 8px;
    font-weight: normal;
}

.wpmm-gdpr-consent label {
    display: flex;
    align-items: flex-start;
    gap: 8px;
    font-weight: normal;
    font-size: 14px;
}

/* Links */
.wpmm-form-links {
    margin-top: 20px;
    text-align: center;
    padding-top: 20px;
    border-top: 1px solid #eee;
}

.wpmm-form-links a {
    display: inline-block;
    margin: 0 10px;
    color: #0073aa;
    text-decoration: none;
    font-size: 14px;
}

.wpmm-form-links a:hover {
    text-decoration: underline;
}

/* Password Requirements */
.wpmm-password-requirements {
    background: #f8f9fa;
    padding: 15px;
    margin-bottom: 20px;
    border-radius: 4px;
    font-size: 13px;
}

.wpmm-password-requirements p {
    margin: 0 0 8px 0;
    font-weight: 600;
    color: #555;
}

.wpmm-password-requirements ul {
    margin: 0;
    padding-left: 20px;
    list-style-type: disc;
}

.wpmm-password-requirements li {
    margin-bottom: 4px;
    color: #666;
}

/* Honeypot */
.wpmm-honeypot {
    position: absolute;
    left: -9999px;
}

/* Dashboard */
.wpmm-dashboard {
    max-width: 800px;
    margin: 40px auto;
    padding: 0 15px;
}

.wpmm-dashboard-header {
    background: #fff;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    margin-bottom: 30px;
}

.wpmm-dashboard-header h1 {
    margin: 0 0 15px 0;
    color: #333;
}

.wpmm-dashboard-nav {
    background: #fff;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    margin-bottom: 30px;
}

.wpmm-dashboard-nav ul {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    gap: 20px;
}

.wpmm-dashboard-nav a {
    color: #0073aa;
    text-decoration: none;
    padding: 10px 15px;
    border-radius: 4px;
    transition: background-color 0.3s;
}

.wpmm-dashboard-nav a:hover {
    background-color: #f0f0f0;
}

.wpmm-dashboard-content {
    background: #fff;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.wpmm-dashboard-section {
    margin-bottom: 30px;
}

.wpmm-dashboard-section h3 {
    margin-top: 0;
    padding-bottom: 10px;
    border-bottom: 1px solid #eee;
    color: #333;
}

/* Profile Form */
.wpmm-profile-edit {
    max-width: 800px;
    margin: 40px auto;
    padding: 0 15px;
}

/* Responsive */
@media (max-width: 768px) {
    .wpmm-form-wrapper {
        padding: 20px;
    }
    
    .wpmm-form-row {
        flex-direction: column;
        gap: 0;
    }
    
    .wpmm-dashboard-nav ul {
        flex-direction: column;
    }
}

/* Loading Spinner */
.wpmm-loading {
    display: inline-block;
    width: 20px;
    height: 20px;
    border: 3px solid rgba(0, 0, 0, 0.1);
    border-radius: 50%;
    border-top-color: #0073aa;
    animation: wpmm-spin 1s ease-in-out infinite;
}

@keyframes wpmm-spin {
    to { transform: rotate(360deg); }
}
```

## 9. FRONTEND JAVASCRIPT - assets/js/frontend.js

```javascript
/**
 * WP Member Manager - Frontend JavaScript
 */
(function($) {
    'use strict';
    
    var WPMM_Frontend = {
        
        /**
         * Initialize frontend functionality
         */
        init: function() {
            this.cacheElements();
            this.bindEvents();
        },
        
        /**
         * Cache DOM elements
         */
        cacheElements: function() {
            this.loginForm = $('#wpmm-login-form');
            this.registerForm = $('#wpmm-register-form');
            this.profileForm = $('#wpmm-profile-form');
            this.messageBox = $('.wpmm-message');
        },
        
        /**
         * Bind events
         */
        bindEvents: function() {
            // Login form submission
            if (this.loginForm.length) {
                this.loginForm.on('submit', this.handleLogin.bind(this));
            }
            
            // Register form submission
            if (this.registerForm.length) {
                this.registerForm.on('submit', this.handleRegister.bind(this));
            }
            
            // Profile form submission
            if (this.profileForm.length) {
                this.profileForm.on('submit', this.handleProfileUpdate.bind(this));
            }
            
            // Logout button
            $('.wpmm-logout-btn').on('click', this.handleLogout.bind(this));
            
            // Password strength meter
            $('#wpmm-reg-password').on('keyup', this.checkPasswordStrength.bind(this));
            
            // Real-time validation
            $('.wpmm-input').on('blur', this.validateField.bind(this));
        },
        
        /**
         * Handle login form submission
         */
        handleLogin: function(e) {
            e.preventDefault();
            
            var self = this;
            var form = $(e.target);
            var submitBtn = form.find('button[type="submit"]');
            var formData = new FormData(form[0]);
            
            // Clear previous messages
            this.clearMessages();
            
            // Basic validation
            if (!this.validateLoginForm(form)) {
                return false;
            }
            
            // Show loading state
            submitBtn.prop('disabled', true).html('<span class="wpmm-loading"></span> ' + 'Logging in...');
            
            // Add action
            formData.append('action', 'wpmm_login');
            formData.append('nonce', wpmm_ajax.nonce);
            
            // Send AJAX request
            $.ajax({
                url: wpmm_ajax.ajax_url,
                type: 'POST',
                data: formData,
                processData: false,
                contentType: false,
                success: function(response) {
                    if (response.success) {
                        self.showMessage(response.data.message, 'success');
                        
                        // Redirect after short delay
                        setTimeout(function() {
                            var redirect = form.find('input[name="redirect"]').val();
                            window.location.href = redirect || wpmm_ajax.dashboard_url;
                        }, 1000);
                    } else {
                        self.showMessage(response.data.message, 'error');
                        submitBtn.prop('disabled', false).html('Login');
                    }
                },
                error: function() {
                    self.showMessage('An error occurred. Please try again.', 'error');
                    submitBtn.prop('disabled', false).html('Login');
                }
            });
        },
        
        /**
         * Handle registration form submission
         */
        handleRegister: function(e) {
            e.preventDefault();
            
            var self = this;
            var form = $(e.target);
            var submitBtn = form.find('button[type="submit"]');
            var formData = new FormData(form[0]);
            
            // Clear previous messages
            this.clearMessages();
            
            // Honeypot check
            if (form.find('input[name="website"]').val()) {
                this.showMessage('Spam detected. Please try again.', 'error');
                return false;
            }
            
            // Validate registration form
            if (!this.validateRegisterForm(form)) {
                return false;
            }
            
            // Show loading state
            submitBtn.prop('disabled', true).html('<span class="wpmm-loading"></span> Creating Account...');
            
            // Add action
            formData.append('action', 'wpmm_register');
            formData.append('nonce', wpmm_ajax.nonce);
            
            // Send AJAX request
            $.ajax({
                url: wpmm_ajax.ajax_url,
                type: 'POST',
                data: formData,
                processData: false,
                contentType: false,
                success: function(response) {
                    if (response.success) {
                        self.showMessage(response.data.message, 'success');
                        form[0].reset();
                        
                        // Redirect to login if email verification not required
                        if (response.data.auto_login) {
                            window.location.href = wpmm_ajax.dashboard_url;
                        } else {
                            submitBtn.prop('disabled', false).html('Create Account');
                        }
                    } else {
                        self.showMessage(response.data.message, 'error');
                        
                        // Show field-specific errors
                        if (response.data.errors) {
                            self.showFieldErrors(response.data.errors);
                        }
                        
                        submitBtn.prop('disabled', false).html('Create Account');
                    }
                },
                error: function() {
                    self.showMessage('An error occurred. Please try again.', 'error');
                    submitBtn.prop('disabled', false).html('Create Account');
                }
            });
        },
        
        /**
         * Handle profile update
         */
        handleProfileUpdate: function(e) {
            e.preventDefault();
            
            var self = this;
            var form = $(e.target);
            var submitBtn = form.find('button[type="submit"]');
            var formData = new FormData(form[0]);
            
            this.clearMessages();
            
            submitBtn.prop('disabled', true).html('Saving...');
            
            formData.append('action', 'wpmm_update_profile');
            formData.append('nonce', wpmm_ajax.nonce);
            
            $.ajax({
                url: wpmm_ajax.ajax_url,
                type: 'POST',
                data: formData,
                processData: false,
                contentType: false,
                success: function(response) {
                    if (response.success) {
                        self.showMessage(response.data.message, 'success');
                    } else {
                        self.showMessage(response.data.message, 'error');
                    }
                    submitBtn.prop('disabled', false).html('Save Changes');
                },
                error: function() {
                    self.showMessage('An error occurred. Please try again.', 'error');
                    submitBtn.prop('disabled', false).html('Save Changes');
                }
            });
        },
        
        /**
         * Handle logout
         */
        handleLogout: function(e) {
            e.preventDefault();
            
            $.ajax({
                url: wpmm_ajax.ajax_url,
                type: 'POST',
                data: {
                    action: 'wpmm_logout',
                    nonce: wpmm_ajax.nonce
                },
                success: function(response) {
                    window.location.href = wpmm_ajax.login_url;
                }
            });
        },
        
        /**
         * Validate login form
         */
        validateLoginForm: function(form) {
            var username = form.find('#wpmm-username').val();
            var password = form.find('#wpmm-password').val();
            var isValid = true;
            
            this.clearFieldErrors();
            
            if (!username) {
                this.showFieldError('wpmm-username', wpmm_ajax.messages.required_field);
                isValid = false;
            }
            
            if (!password) {
                this.showFieldError('wpmm-password', wpmm_ajax.messages.required_field);
                isValid = false;
            }
            
            return isValid;
        },
        
        /**
         * Validate register form
         */
        validateRegisterForm: function(form) {
            var self = this;
            var isValid = true;
            
            this.clearFieldErrors();
            
            var validations = [
                { field: 'wpmm-first-name', value: form.find('#wpmm-first-name').val(), message: 'First name is required' },
                { field: 'wpmm-last-name', value: form.find('#wpmm-last-name').val(), message: 'Last name is required' },
                { field: 'wpmm-reg-username', value: form.find('#wpmm-reg-username').val(), message: 'Username is required' },
                { field: 'wpmm-reg-email', value: form.find('#wpmm-reg-email').val(), message: 'Valid email is required' },
                { field: 'wpmm-reg-password', value: form.find('#wpmm-reg-password').val(), message: 'Password is required' },
                { field: 'wpmm-reg-confirm-password', value: form.find('#wpmm-reg-confirm-password').val(), message: 'Please confirm password' }
            ];
            
            $.each(validations, function(index, validation) {
                if (!validation.value) {
                    self.showFieldError(validation.field, validation.message);
                    isValid = false;
                }
            });
            
            // Check password match
            var password = form.find('#wpmm-reg-password').val();
            var confirmPassword = form.find('#wpmm-reg-confirm-password').val();
            
            if (password && confirmPassword && password !== confirmPassword) {
                self.showFieldError('wpmm-reg-confirm-password', wpmm_ajax.messages.password_mismatch);
                isValid = false;
            }
            
            // Check password strength
            if (password && !this.isStrongPassword(password)) {
                self.showFieldError('wpmm-reg-password', wpmm_ajax.messages.password_strength);
                isValid = false;
            }
            
            // Email validation
            var email = form.find('#wpmm-reg-email').val();
            if (email && !this.isValidEmail(email)) {
                self.showFieldError('wpmm-reg-email', wpmm_ajax.messages.invalid_email);
                isValid = false;
            }
            
            return isValid;
        },
        
        /**
         * Check password strength
         */
        checkPasswordStrength: function() {
            var password = $('#wpmm-reg-password').val();
            var requirements = $('.wpmm-password-requirements li');
            
            var checks = {
                0: /.{8,}/,
                1: /[A-Z]/,
                2: /[a-z]/,
                3: /[0-9]/,
                4: /[!@#$%^&*]/
            };
            
            requirements.each(function(index) {
                if (checks[index] && checks[index].test(password)) {
                    $(this).css('color', '#28a745');
                } else {
                    $(this).css('color', '#666');
                }
            });
        },
        
        /**
         * Check if password is strong
         */
        isStrongPassword: function(password) {
            return /.{8,}/.test(password) &&
                   /[A-Z]/.test(password) &&
                   /[a-z]/.test(password) &&
                   /[0-9]/.test(password) &&
                   /[!@#$%^&*]/.test(password);
        },
        
        /**
         * Validate email format
         */
        isValidEmail: function(email) {
            return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
        },
        
        /**
         * Validate individual field
         */
        validateField: function(e) {
            var field = $(e.target);
            var fieldId = field.attr('id');
            
            // Remove error if field has value
            if (field.val()) {
                this.clearFieldError(fieldId);
            }
        },
        
        /**
         * Show field error
         */
        showFieldError: function(fieldId, message) {
            $('#' + fieldId).addClass('error');
            $('#' + fieldId).after('<span class="wpmm-field-error">' + message + '</span>');
        },
        
        /**
         * Show field errors
         */
        showFieldErrors: function(errors) {
            var self = this;
            $.each(errors, function(field, message) {
                self.showFieldError(field, message);
            });
        },
        
        /**
         * Clear all field errors
         */
        clearFieldErrors: function() {
            $('.wpmm-field-error').remove();
            $('.wpmm-input').removeClass('error');
        },
        
        /**
         * Clear field error
         */
        clearFieldError: function(fieldId) {
            $('#' + fieldId).removeClass('error');
            $('#' + fieldId).next('.wpmm-field-error').remove();
        },
        
        /**
         * Show message
         */
        showMessage: function(message, type) {
            this.messageBox
                .removeClass('wpmm-message-success wpmm-message-error wpmm-message-info')
                .addClass('wpmm-message-' + type)
                .html(message)
                .fadeIn();
        },
        
        /**
         * Clear messages
         */
        clearMessages: function() {
            this.messageBox
                .removeClass('wpmm-message-success wpmm-message-error wpmm-message-info')
                .html('')
                .hide();
        }
    };
    
    // Initialize on document ready
    $(document).ready(function() {
        WPMM_Frontend.init();
    });
    
})(jQuery);
```

## 10. SHORTCODE HANDLING - Add to class-member-hooks.php

```php
<?php
/**
 * Member Hooks & Shortcodes
 */

if (!defined('ABSPATH')) {
    exit;
}

class WP_Member_Hooks {
    
    private $plugin;
    
    public function __construct($plugin) {
        $this->plugin = $plugin;
        $this->register_hooks();
    }
    
    /**
     * Register all hooks and shortcodes
     */
    private function register_hooks() {
        // Shortcodes
        add_shortcode('wpmm_login_form', array($this, 'shortcode_login_form'));
        add_shortcode('wpmm_register_form', array($this, 'shortcode_register_form'));
        add_shortcode('wpmm_dashboard', array($this, 'shortcode_dashboard'));
        add_shortcode('wpmm_profile_edit', array($this, 'shortcode_profile_edit'));
        add_shortcode('wpmm_restricted', array($this, 'shortcode_restricted_content'));
        
        // Member ID filter for other plugins
        add_filter('wpmm_get_member_id', array($this, 'provide_member_id'), 10, 2);
        add_filter('wpmm_is_logged_in', array($this, 'check_login_status'));
        add_filter('wpmm_get_current_member', array($this, 'get_current_member_data'));
        
        // Content restriction
        add_filter('the_content', array($this, 'restrict_content'));
    }
    
    /**
     * Login form shortcode
     */
    public function shortcode_login_form() {
        // Don't show login form if already logged in
        if ($this->plugin->get_component('session')->is_logged_in()) {
            $dashboard_url = $this->plugin->get_page_url('dashboard');
            return '<p>' . __('You are already logged in.', 'wp-member-manager') . 
                   ' <a href="' . esc_url($dashboard_url) . '">' . 
                   __('Go to Dashboard', 'wp-member-manager') . '</a></p>';
        }
        
        return WPMM_Template_Loader::get_template('login-form.php');
    }
    
    /**
     * Registration form shortcode
     */
    public function shortcode_register_form() {
        if ($this->plugin->get_component('session')->is_logged_in()) {
            $dashboard_url = $this->plugin->get_page_url('dashboard');
            return '<p>' . __('You already have an account.', 'wp-member-manager') . 
                   ' <a href="' . esc_url($dashboard_url) . '">' . 
                   __('Go to Dashboard', 'wp-member-manager') . '</a></p>';
        }
        
        return WPMM_Template_Loader::get_template('register-form.php');
    }
    
    /**
     * Dashboard shortcode
     */
    public function shortcode_dashboard() {
        $session = $this->plugin->get_component('session');
        
        if (!$session->is_logged_in()) {
            return $this->shortcode_login_form();
        }
        
        $member = $session->get_current_member();
        $profile_url = $this->plugin->get_page_url('profile');
        
        ob_start();
        ?>
        <div class="wpmm-dashboard">
            <div class="wpmm-dashboard-header">
                <h1><?php printf(__('Welcome, %s!', 'wp-member-manager'), esc_html($member['display_name'])); ?></h1>
                <p>
                    <?php _e('Member ID:', 'wp-member-manager'); ?> 
                    <code><?php echo esc_html($member['member_id']); ?></code>
                </p>
                <p>
                    <?php _e('Membership Level:', 'wp-member-manager'); ?> 
                    <strong><?php echo esc_html(ucfirst($member['membership_level'])); ?></strong>
                </p>
                <p>
                    <?php _e('Member since:', 'wp-member-manager'); ?> 
                    <?php echo date_i18n(get_option('date_format'), strtotime($member['registration_date'])); ?>
                </p>
            </div>
            
            <div class="wpmm-dashboard-nav">
                <ul>
                    <li><a href="<?php echo esc_url($profile_url); ?>"><?php _e('Edit Profile', 'wp-member-manager'); ?></a></li>
                    <li><a href="#" class="wpmm-logout-btn"><?php _e('Logout', 'wp-member-manager'); ?></a></li>
                </ul>
            </div>
            
            <div class="wpmm-dashboard-content">
                <div class="wpmm-dashboard-section">
                    <h3><?php _e('Account Information', 'wp-member-manager'); ?></h3>
                    <table class="wpmm-info-table">
                        <tr>
                            <td><strong><?php _e('Name:', 'wp-member-manager'); ?></strong></td>
                            <td><?php echo esc_html($member['first_name'] . ' ' . $member['last_name']); ?></td>
                        </tr>
                        <tr>
                            <td><strong><?php _e('Email:', 'wp-member-manager'); ?></strong></td>
                            <td><?php echo esc_html($member['email']); ?></td>
                        </tr>
                        <tr>
                            <td><strong><?php _e('Username:', 'wp-member-manager'); ?></strong></td>
                            <td><?php echo esc_html($member['username']); ?></td>
                        </tr>
                        <tr>
                            <td><strong><?php _e('Status:', 'wp-member-manager'); ?></strong></td>
                            <td><span class="wpmm-status wpmm-status-<?php echo esc_attr($member['status']); ?>">
                                <?php echo esc_html(ucfirst($member['status'])); ?>
                            </span></td>
                        </tr>
                    </table>
                </div>
                
                <!-- Additional dashboard sections can be added by other plugins -->
                <?php do_action('wpmm_dashboard_content', $member['member_id']); ?>
            </div>
        </div>
        <?php
        return ob_get_clean();
    }
    
    /**
     * Profile edit shortcode
     */
    public function shortcode_profile_edit() {
        $session = $this->plugin->get_component('session');
        
        if (!$session->is_logged_in()) {
            return $this->shortcode_login_form();
        }
        
        return WPMM_Template_Loader::get_template('profile-edit.php', array(
            'member' => $session->get_current_member()
        ));
    }
    
    /**
     * Restricted content shortcode
     */
    public function shortcode_restricted_content($atts, $content = null) {
        $atts = shortcode_atts(array(
            'level' => 'free',
            'message' => ''
        ), $atts);
        
        if ($this->plugin->get_component('session')->is_logged_in()) {
            $member = $this->plugin->get_component('session')->get_current_member();
            
            if ($this->check_membership_level($member, $atts['level'])) {
                return do_shortcode($content);
            }
        }
        
        if (!empty($atts['message'])) {
            return '<div class="wpmm-restricted-message">' . esc_html($atts['message']) . '</div>';
        }
        
        return '<div class="wpmm-restricted-message">' . 
               sprintf(__('This content is available for %s members only.', 'wp-member-manager'), 
               ucfirst($atts['level'])) . 
               ' <a href="' . esc_url($this->plugin->get_page_url('login')) . '">' . 
               __('Login', 'wp-member-manager') . '</a></div>';
    }
    
    /**
     * Check membership level
     */
    private function check_membership_level($member, $required_level) {
        $levels = array('free' => 0, 'basic' => 1, 'premium' => 2, 'vip' => 3, 'admin' => 4);
        
        $member_level = $levels[$member['membership_level']] ?? 0;
        $required = $levels[$required_level] ?? 0;
        
        return $member_level >= $required;
    }
    
    /**
     * Provide member ID to other plugins
     */
    public function provide_member_id($member_id = null, $args = array()) {
        if ($member_id) {
            return $member_id;
        }
        
        $session = $this->plugin->get_component('session');
        
        if ($session->is_logged_in()) {
            return $session->get_current_member_id();
        }
        
        return null;
    }
    
    /**
     * Check login status
     */
    public function check_login_status() {
        return $this->plugin->get_component('session')->is_logged_in();
    }
    
    /**
     * Get current member data
     */
    public function get_current_member_data() {
        return $this->plugin->get_component('session')->get_current_member();
    }
    
    /**
     * Restrict content based on membership
     */
    public function restrict_content($content) {
        // Only restrict if post has membership restriction
        $required_level = get_post_meta(get_the_ID(), '_wpmm_required_level', true);
        
        if (!$required_level) {
            return $content;
        }
        
        if ($this->plugin->get_component('session')->is_logged_in()) {
            $member = $this->plugin->get_component('session')->get_current_member();
            
            if ($this->check_membership_level($member, $required_level)) {
                return $content;
            }
        }
        
        $restricted_message = '<div class="wpmm-content-restricted">';
        $restricted_message .= '<h3>' . __('Content Restricted', 'wp-member-manager') . '</h3>';
        $restricted_message .= '<p>' . sprintf(
            __('This content is available for %s members only.', 'wp-member-manager'),
            ucfirst($required_level)
        ) . '</p>';
        
        if (!$this->plugin->get_component('session')->is_logged_in()) {
            $restricted_message .= '<p><a href="' . esc_url($this->plugin->get_page_url('login')) . '" class="wpmm-button wpmm-button-primary">' . 
                                  __('Login to Access', 'wp-member-manager') . '</a></p>';
        } else {
            $restricted_message .= '<p>' . __('Upgrade your membership to access this content.', 'wp-member-manager') . '</p>';
        }
        
        $restricted_message .= '</div>';
        
        return $restricted_message;
    }
}
```

This complete WordPress Member Manager plugin provides:

✅ **Complete member management** - Registration, login, profile editing, dashboard
✅ **Secure authentication** - Password hashing, session management, brute force protection
✅ **AJAX-powered forms** - Smooth user experience without page reloads
✅ **Extensible hook system** - Other plugins can easily integrate
✅ **Member ID sharing** - Consistent user IDs across plugins
✅ **Content restriction** - Shortcode and post-level content protection
✅ **Responsive design** - Works on all devices
✅ **Shared hosting ready** - Uses WordPress standards and minimal requirements

To install, simply upload the entire `wp-member-manager` folder to your `/wp-content/plugins/` directory and activate from the WordPress admin panel.
