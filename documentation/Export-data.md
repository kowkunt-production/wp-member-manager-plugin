# Export data

A comprehensive solutions to export all members 
from the WP Member Manager plugin database. 

With multiple export methods including 
admin interface, CSV export, and direct database queries.

## 1. ADD EXPORT FUNCTIONALITY TO WP MEMBER MANAGER

### includes/class-member-export.php

```php
<?php
/**
 * Member Export Functionality
 * Export members to CSV, JSON, or XML
 */

if (!defined('ABSPATH')) {
    exit;
}

class WP_Member_Export {
    
    private $wpdb;
    private $members_table;
    private $meta_table;
    
    public function __construct() {
        global $wpdb;
        $this->wpdb = $wpdb;
        $this->members_table = WPMM_MEMBERS_TABLE;
        $this->meta_table = WPMM_META_TABLE;
        
        // Register AJAX handlers
        add_action('wp_ajax_wpmm_export_members', array($this, 'ajax_export_members'));
        add_action('wp_ajax_wpmm_download_export', array($this, 'ajax_download_export'));
        
        // Admin page for export
        add_action('admin_menu', array($this, 'add_export_page'));
    }
    
    /**
     * Add export page to admin menu
     */
    public function add_export_page() {
        add_submenu_page(
            'wp-member-manager',
            __('Export Members', 'wp-member-manager'),
            __('Export Members', 'wp-member-manager'),
            'manage_options',
            'wpmm-export',
            array($this, 'render_export_page')
        );
    }
    
    /**
     * Render export page
     */
    public function render_export_page() {
        ?>
        <div class="wrap wpmm-export-page">
            <h1><?php _e('Export Members', 'wp-member-manager'); ?></h1>
            
            <div class="wpmm-export-container">
                <!-- Export Options -->
                <div class="wpmm-card">
                    <h2><?php _e('Export Options', 'wp-member-manager'); ?></h2>
                    
                    <form id="wpmm-export-form" method="post">
                        <?php wp_nonce_field('wpmm_export_nonce', 'wpmm_export_nonce'); ?>
                        
                        <table class="form-table">
                            <tr>
                                <th scope="row">
                                    <label for="export_format"><?php _e('Export Format', 'wp-member-manager'); ?></label>
                                </th>
                                <td>
                                    <select name="export_format" id="export_format">
                                        <option value="csv">CSV (Comma Separated Values)</option>
                                        <option value="json">JSON</option>
                                        <option value="xml">XML</option>
                                        <option value="excel">Excel (XLSX compatible CSV)</option>
                                    </select>
                                </td>
                            </tr>
                            
                            <tr>
                                <th scope="row">
                                    <label for="export_status"><?php _e('Member Status', 'wp-member-manager'); ?></label>
                                </th>
                                <td>
                                    <select name="export_status" id="export_status">
                                        <option value="all">All Members</option>
                                        <option value="active">Active Only</option>
                                        <option value="pending">Pending Only</option>
                                        <option value="suspended">Suspended Only</option>
                                        <option value="deleted">Deleted Only</option>
                                    </select>
                                </td>
                            </tr>
                            
                            <tr>
                                <th scope="row">
                                    <label for="export_membership"><?php _e('Membership Level', 'wp-member-manager'); ?></label>
                                </th>
                                <td>
                                    <select name="export_membership" id="export_membership">
                                        <option value="all">All Levels</option>
                                        <option value="free">Free</option>
                                        <option value="basic">Basic</option>
                                        <option value="premium">Premium</option>
                                        <option value="vip">VIP</option>
                                    </select>
                                </td>
                            </tr>
                            
                            <tr>
                                <th scope="row">
                                    <label for="export_date_from"><?php _e('Registration Date Range', 'wp-member-manager'); ?></label>
                                </th>
                                <td>
                                    <input type="date" name="export_date_from" id="export_date_from">
                                    <?php _e('to', 'wp-member-manager'); ?>
                                    <input type="date" name="export_date_to" id="export_date_to">
                                </td>
                            </tr>
                            
                            <tr>
                                <th scope="row">
                                    <label><?php _e('Fields to Export', 'wp-member-manager'); ?></label>
                                </th>
                                <td>
                                    <fieldset>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="member_id" checked>
                                            <?php _e('Member ID', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="username" checked>
                                            <?php _e('Username', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="email" checked>
                                            <?php _e('Email', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="first_name" checked>
                                            <?php _e('First Name', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="last_name" checked>
                                            <?php _e('Last Name', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="display_name" checked>
                                            <?php _e('Display Name', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="membership_level" checked>
                                            <?php _e('Membership Level', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="status" checked>
                                            <?php _e('Status', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="registration_date" checked>
                                            <?php _e('Registration Date', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="last_login">
                                            <?php _e('Last Login', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="email_verified">
                                            <?php _e('Email Verified', 'wp-member-manager'); ?>
                                        </label><br>
                                        <label>
                                            <input type="checkbox" name="export_fields[]" value="meta">
                                            <?php _e('Include Custom Meta Fields', 'wp-member-manager'); ?>
                                        </label>
                                    </fieldset>
                                </td>
                            </tr>
                            
                            <tr>
                                <th scope="row">
                                    <label for="export_limit"><?php _e('Limit', 'wp-member-manager'); ?></label>
                                </th>
                                <td>
                                    <input type="number" name="export_limit" id="export_limit" 
                                           value="0" min="0" step="100">
                                    <p class="description">
                                        <?php _e('Enter 0 to export all members. Use this for large exports.', 'wp-member-manager'); ?>
                                    </p>
                                </td>
                            </tr>
                            
                            <tr>
                                <th scope="row">
                                    <label for="export_offset"><?php _e('Offset', 'wp-member-manager'); ?></label>
                                </th>
                                <td>
                                    <input type="number" name="export_offset" id="export_offset" 
                                           value="0" min="0" step="100">
                                    <p class="description">
                                        <?php _e('Start from this record number. Useful for batch exports.', 'wp-member-manager'); ?>
                                    </p>
                                </td>
                            </tr>
                        </table>
                        
                        <p class="submit">
                            <button type="submit" class="button button-primary" id="wpmm-export-btn">
                                <?php _e('Export Members', 'wp-member-manager'); ?>
                            </button>
                            <button type="button" class="button" id="wpmm-preview-btn">
                                <?php _e('Preview Data', 'wp-member-manager'); ?>
                            </button>
                            <button type="button" class="button" id="wpmm-count-btn">
                                <?php _e('Count Members', 'wp-member-manager'); ?>
                            </button>
                        </p>
                    </form>
                </div>
                
                <!-- Preview Area -->
                <div class="wpmm-card" id="wpmm-preview-area" style="display:none;">
                    <h2><?php _e('Preview', 'wp-member-manager'); ?></h2>
                    <div id="wpmm-preview-content"></div>
                </div>
                
                <!-- Export History -->
                <div class="wpmm-card">
                    <h2><?php _e('Export History', 'wp-member-manager'); ?></h2>
                    <div id="wpmm-export-history">
                        <?php $this->render_export_history(); ?>
                    </div>
                </div>
                
                <!-- Quick Export Buttons -->
                <div class="wpmm-card">
                    <h2><?php _e('Quick Exports', 'wp-member-manager'); ?></h2>
                    <p><?php _e('One-click exports with default settings:', 'wp-member-manager'); ?></p>
                    
                    <button type="button" class="button quick-export" data-format="csv" data-status="active">
                        <?php _e('Export Active Members (CSV)', 'wp-member-manager'); ?>
                    </button>
                    
                    <button type="button" class="button quick-export" data-format="json" data-status="all">
                        <?php _e('Export All Members (JSON)', 'wp-member-manager'); ?>
                    </button>
                    
                    <button type="button" class="button quick-export" data-format="excel" data-status="all">
                        <?php _e('Export All Members (Excel)', 'wp-member-manager'); ?>
                    </button>
                </div>
            </div>
        </div>
        
        <style>
        .wpmm-export-container {
            max-width: 1200px;
            margin-top: 20px;
        }
        .wpmm-card {
            background: #fff;
            border: 1px solid #ccd0d4;
            border-radius: 4px;
            padding: 20px;
            margin-bottom: 20px;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        .wpmm-card h2 {
            margin-top: 0;
            padding-bottom: 10px;
            border-bottom: 1px solid #eee;
        }
        .quick-export {
            margin-right: 10px !important;
            margin-bottom: 10px !important;
        }
        .wpmm-loading {
            display: inline-block;
            width: 20px;
            height: 20px;
            border: 3px solid rgba(0,0,0,0.1);
            border-radius: 50%;
            border-top-color: #0073aa;
            animation: wpmm-spin 1s ease-in-out infinite;
            vertical-align: middle;
            margin-left: 10px;
        }
        @keyframes wpmm-spin {
            to { transform: rotate(360deg); }
        }
        .wpmm-export-progress {
            margin-top: 10px;
            padding: 10px;
            background: #f0f6fc;
            border-left: 4px solid #0073aa;
            display: none;
        }
        .wpmm-progress-bar {
            height: 20px;
            background: #e0e0e0;
            border-radius: 10px;
            overflow: hidden;
            margin: 10px 0;
        }
        .wpmm-progress-fill {
            height: 100%;
            background: #0073aa;
            width: 0%;
            transition: width 0.3s ease;
        }
        </style>
        
        <script>
        jQuery(document).ready(function($) {
            // Handle export form submission
            $('#wpmm-export-form').on('submit', function(e) {
                e.preventDefault();
                exportMembers('download');
            });
            
            // Preview button
            $('#wpmm-preview-btn').on('click', function() {
                exportMembers('preview');
            });
            
            // Count button
            $('#wpmm-count-btn').on('click', function() {
                countMembers();
            });
            
            // Quick export buttons
            $('.quick-export').on('click', function() {
                var format = $(this).data('format');
                var status = $(this).data('status');
                
                $('#export_format').val(format);
                $('#export_status').val(status);
                
                exportMembers('download');
            });
            
            function exportMembers(action) {
                var formData = $('#wpmm-export-form').serialize();
                formData += '&action=wpmm_export_members&export_action=' + action;
                formData += '&nonce=' + $('#wpmm_export_nonce').val();
                
                var $btn = $('#wpmm-export-btn');
                var $progress = $('.wpmm-export-progress');
                
                if (action === 'download') {
                    $btn.prop('disabled', true).html('Exporting... <span class="wpmm-loading"></span>');
                    $progress.show();
                }
                
                $.ajax({
                    url: ajaxurl,
                    type: 'POST',
                    data: formData,
                    xhrFields: {
                        responseType: action === 'download' ? 'blob' : 'text'
                    },
                    success: function(response, status, xhr) {
                        if (action === 'download') {
                            // Get filename from headers
                            var filename = 'members-export.csv';
                            var disposition = xhr.getResponseHeader('Content-Disposition');
                            if (disposition && disposition.indexOf('attachment') !== -1) {
                                var filenameRegex = /filename[^;=\n]*=((['"]).*?\2|[^;\n]*)/;
                                var matches = filenameRegex.exec(disposition);
                                if (matches != null && matches[1]) { 
                                    filename = matches[1].replace(/['"]/g, '');
                                }
                            }
                            
                            // Download file
                            var blob = new Blob([response], { type: xhr.getResponseHeader('Content-Type') });
                            var link = document.createElement('a');
                            link.href = window.URL.createObjectURL(blob);
                            link.download = filename;
                            link.click();
                            
                            $btn.prop('disabled', false).html('Export Members');
                            $progress.hide();
                            
                            // Refresh export history
                            refreshExportHistory();
                            
                            alert('Export completed successfully!');
                        } else {
                            // Preview
                            $('#wpmm-preview-area').show();
                            $('#wpmm-preview-content').html(response);
                        }
                    },
                    error: function(xhr, status, error) {
                        alert('Export failed: ' + error);
                        $btn.prop('disabled', false).html('Export Members');
                        $progress.hide();
                    }
                });
            }
            
            function countMembers() {
                var formData = $('#wpmm-export-form').serialize();
                formData += '&action=wpmm_export_members&export_action=count';
                formData += '&nonce=' + $('#wpmm_export_nonce').val();
                
                $.ajax({
                    url: ajaxurl,
                    type: 'POST',
                    data: formData,
                    success: function(response) {
                        var data = JSON.parse(response);
                        alert('Total members matching criteria: ' + data.count);
                    }
                });
            }
            
            function refreshExportHistory() {
                $.ajax({
                    url: ajaxurl,
                    type: 'POST',
                    data: {
                        action: 'wpmm_export_members',
                        export_action: 'history',
                        nonce: $('#wpmm_export_nonce').val()
                    },
                    success: function(response) {
                        $('#wpmm-export-history').html(response);
                    }
                });
            }
        });
        </script>
        <?php
    }
    
    /**
     * AJAX handler for export
     */
    public function ajax_export_members() {
        // Verify nonce
        if (!wp_verify_nonce($_POST['nonce'], 'wpmm_export_nonce')) {
            wp_die('Security check failed');
        }
        
        // Check permissions
        if (!current_user_can('manage_options')) {
            wp_die('Insufficient permissions');
        }
        
        $action = $_POST['export_action'] ?? 'download';
        
        switch ($action) {
            case 'count':
                $this->count_members();
                break;
            case 'preview':
                $this->preview_members();
                break;
            case 'history':
                $this->render_export_history();
                break;
            case 'download':
            default:
                $this->download_export();
                break;
        }
        
        wp_die();
    }
    
    /**
     * Get members based on filters
     */
    private function get_filtered_members() {
        $status = sanitize_text_field($_POST['export_status'] ?? 'all');
        $membership = sanitize_text_field($_POST['export_membership'] ?? 'all');
        $date_from = sanitize_text_field($_POST['export_date_from'] ?? '');
        $date_to = sanitize_text_field($_POST['export_date_to'] ?? '');
        $limit = intval($_POST['export_limit'] ?? 0);
        $offset = intval($_POST['export_offset'] ?? 0);
        
        $where = array('1=1');
        
        // Status filter
        if ($status !== 'all') {
            $where[] = $this->wpdb->prepare('m.status = %s', $status);
        } else {
            $where[] = "m.status != 'deleted'";
        }
        
        // Membership filter
        if ($membership !== 'all') {
            $where[] = $this->wpdb->prepare('m.membership_level = %s', $membership);
        }
        
        // Date range filter
        if (!empty($date_from)) {
            $where[] = $this->wpdb->prepare('m.registration_date >= %s', $date_from . ' 00:00:00');
        }
        if (!empty($date_to)) {
            $where[] = $this->wpdb->prepare('m.registration_date <= %s', $date_to . ' 23:59:59');
        }
        
        $where_clause = implode(' AND ', $where);
        
        // Build query
        $query = "SELECT m.* FROM {$this->members_table} m WHERE {$where_clause} ORDER BY m.registration_date DESC";
        
        if ($limit > 0) {
            $query .= $this->wpdb->prepare(' LIMIT %d OFFSET %d', $limit, $offset);
        }
        
        return $this->wpdb->get_results($query, ARRAY_A);
    }
    
    /**
     * Count members
     */
    private function count_members() {
        $members = $this->get_filtered_members();
        $total = $this->wpdb->get_var("SELECT FOUND_ROWS()");
        
        echo json_encode(array(
            'count' => count($members),
            'total' => $total ?: count($members)
        ));
    }
    
    /**
     * Preview members
     */
    private function preview_members() {
        $_POST['export_limit'] = 10; // Preview only 10 records
        $members = $this->get_filtered_members();
        
        if (empty($members)) {
            echo '<p>' . __('No members found matching your criteria.', 'wp-member-manager') . '</p>';
            return;
        }
        
        echo '<table class="wp-list-table widefat fixed striped">';
        echo '<thead><tr>';
        
        // Get selected fields or show all
        $fields = $_POST['export_fields'] ?? array_keys($members[0]);
        
        foreach ($fields as $field) {
            echo '<th>' . esc_html(ucwords(str_replace('_', ' ', $field))) . '</th>';
        }
        
        echo '</tr></thead><tbody>';
        
        foreach ($members as $member) {
            echo '<tr>';
            foreach ($fields as $field) {
                if ($field === 'meta') {
                    $meta = $this->get_member_meta_export($member['member_id']);
                    echo '<td><pre>' . esc_html(json_encode($meta, JSON_PRETTY_PRINT)) . '</pre></td>';
                } else {
                    echo '<td>' . esc_html($member[$field] ?? '') . '</td>';
                }
            }
            echo '</tr>';
        }
        
        echo '</tbody></table>';
        echo '<p><em>' . sprintf(__('Showing first 10 records. Use Export to get all records.', 'wp-member-manager')) . '</em></p>';
    }
    
    /**
     * Download export file
     */
    private function download_export() {
        $format = sanitize_text_field($_POST['export_format'] ?? 'csv');
        $members = $this->get_filtered_members();
        $fields = $_POST['export_fields'] ?? array('member_id', 'username', 'email', 'first_name', 'last_name', 'membership_level', 'status', 'registration_date');
        
        if (empty($members)) {
            wp_die('No members found to export.');
        }
        
        // Add meta data if requested
        if (in_array('meta', $fields)) {
            foreach ($members as &$member) {
                $member['meta'] = json_encode($this->get_member_meta_export($member['member_id']));
            }
        }
        
        // Remove 'meta' from fields array to avoid duplication
        $fields = array_diff($fields, array('meta'));
        
        $filename = 'member-export-' . date('Y-m-d-His');
        
        switch ($format) {
            case 'json':
                $this->export_json($members, $filename);
                break;
            case 'xml':
                $this->export_xml($members, $fields, $filename);
                break;
            case 'excel':
                $this->export_csv($members, $fields, $filename, true);
                break;
            case 'csv':
            default:
                $this->export_csv($members, $fields, $filename);
                break;
        }
        
        // Log export
        $this->log_export($format, count($members));
        
        exit;
    }
    
    /**
     * Export as CSV
     */
    private function export_csv($members, $fields, $filename, $excel = false) {
        $delimiter = $excel ? "\t" : ',';
        $enclosure = '"';
        $extension = $excel ? 'xls' : 'csv';
        $mime_type = $excel ? 'application/vnd.ms-excel' : 'text/csv';
        
        // Set headers for download
        header('Content-Type: ' . $mime_type . '; charset=utf-8');
        header('Content-Disposition: attachment; filename="' . $filename . '.' . $extension . '"');
        header('Pragma: no-cache');
        header('Expires: 0');
        
        // Add BOM for Excel UTF-8 compatibility
        if ($excel) {
            echo "\xEF\xBB\xBF";
        }
        
        // Open output stream
        $output = fopen('php://output', 'w');
        
        // Add headers
        $headers = array();
        foreach ($fields as $field) {
            $headers[] = ucwords(str_replace('_', ' ', $field));
        }
        fputcsv($output, $headers, $delimiter, $enclosure);
        
        // Add data rows
        foreach ($members as $member) {
            $row = array();
            foreach ($fields as $field) {
                $row[] = $member[$field] ?? '';
            }
            fputcsv($output, $row, $delimiter, $enclosure);
        }
        
        fclose($output);
    }
    
    /**
     * Export as JSON
     */
    private function export_json($members, $filename) {
        // Remove sensitive data
        foreach ($members as &$member) {
            unset($member['password_hash']);
            unset($member['login_attempts']);
            unset($member['locked_until']);
        }
        
        header('Content-Type: application/json; charset=utf-8');
        header('Content-Disposition: attachment; filename="' . $filename . '.json"');
        
        echo json_encode($members, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
    }
    
    /**
     * Export as XML
     */
    private function export_xml($members, $fields, $filename) {
        header('Content-Type: application/xml; charset=utf-8');
        header('Content-Disposition: attachment; filename="' . $filename . '.xml"');
        
        $xml = new SimpleXMLElement('<?xml version="1.0" encoding="UTF-8"?><members></members>');
        
        foreach ($members as $member) {
            $member_xml = $xml->addChild('member');
            
            foreach ($fields as $field) {
                // Remove sensitive data
                if (in_array($field, array('password_hash', 'login_attempts', 'locked_until'))) {
                    continue;
                }
                
                $value = $member[$field] ?? '';
                $member_xml->addChild($field, htmlspecialchars($value));
            }
        }
        
        echo $xml->asXML();
    }
    
    /**
     * Get member meta for export
     */
    private function get_member_meta_export($member_id) {
        $meta = $this->wpdb->get_results(
            $this->wpdb->prepare(
                "SELECT meta_key, meta_value FROM {$this->meta_table} WHERE member_id = %s",
                $member_id
            ),
            ARRAY_A
        );
        
        $formatted_meta = array();
        foreach ($meta as $item) {
            // Skip sensitive meta keys
            if (in_array($item['meta_key'], array('verification_token', 'verification_token_expiry', 'reset_token', 'reset_token_expiry'))) {
                continue;
            }
            $formatted_meta[$item['meta_key']] = maybe_unserialize($item['meta_value']);
        }
        
        return $formatted_meta;
    }
    
    /**
     * Log export to history
     */
    private function log_export($format, $count) {
        $history = get_option('wpmm_export_history', array());
        
        $history[] = array(
            'format' => $format,
            'count' => $count,
            'user_id' => get_current_user_id(),
            'user_name' => wp_get_current_user()->display_name,
            'date' => current_time('mysql'),
            'filters' => array(
                'status' => $_POST['export_status'] ?? 'all',
                'membership' => $_POST['export_membership'] ?? 'all',
                'date_from' => $_POST['export_date_from'] ?? '',
                'date_to' => $_POST['export_date_to'] ?? ''
            )
        );
        
        // Keep only last 50 exports
        if (count($history) > 50) {
            $history = array_slice($history, -50);
        }
        
        update_option('wpmm_export_history', $history);
    }
    
    /**
     * Render export history
     */
    private function render_export_history() {
        $history = get_option('wpmm_export_history', array());
        
        if (empty($history)) {
            echo '<p>' . __('No export history yet.', 'wp-member-manager') . '</p>';
            return;
        }
        
        // Show most recent first
        $history = array_reverse($history);
        
        echo '<table class="wp-list-table widefat fixed striped">';
        echo '<thead><tr>';
        echo '<th>' . __('Date', 'wp-member-manager') . '</th>';
        echo '<th>' . __('User', 'wp-member-manager') . '</th>';
        echo '<th>' . __('Format', 'wp-member-manager') . '</th>';
        echo '<th>' . __('Members', 'wp-member-manager') . '</th>';
        echo '<th>' . __('Filters', 'wp-member-manager') . '</th>';
        echo '</tr></thead><tbody>';
        
        foreach ($history as $export) {
            echo '<tr>';
            echo '<td>' . date_i18n(get_option('date_format') . ' ' . get_option('time_format'), strtotime($export['date'])) . '</td>';
            echo '<td>' . esc_html($export['user_name']) . '</td>';
            echo '<td>' . strtoupper(esc_html($export['format'])) . '</td>';
            echo '<td>' . esc_html($export['count']) . '</td>';
            echo '<td>';
            $filters = array();
            if ($export['filters']['status'] !== 'all') {
                $filters[] = 'Status: ' . $export['filters']['status'];
            }
            if ($export['filters']['membership'] !== 'all') {
                $filters[] = 'Level: ' . $export['filters']['membership'];
            }
            if (!empty($export['filters']['date_from'])) {
                $filters[] = 'From: ' . $export['filters']['date_from'];
            }
            if (!empty($export['filters']['date_to'])) {
                $filters[] = 'To: ' . $export['filters']['date_to'];
            }
            echo esc_html(implode(', ', $filters) ?: 'None');
            echo '</td>';
            echo '</tr>';
        }
        
        echo '</tbody></table>';
    }
}

// Initialize export functionality
new WP_Member_Export();
```

## 2. DIRECT DATABASE EXPORT SCRIPTS

### export-members-direct.php (Standalone Script)

```php
<?php
/**
 * Direct Database Export Script
 * 
 * Upload this file to your WordPress root directory and access via browser
 * WARNING: Delete this file after use for security reasons!
 * 
 * Usage: https://yoursite.com/export-members-direct.php?format=csv&key=YOUR_SECRET_KEY
 */

// Set your secret key here
define('EXPORT_SECRET_KEY', 'your-secret-key-here-change-this');

// Load WordPress
require_once('wp-load.php');

// Verify secret key
if (!isset($_GET['key']) || $_GET['key'] !== EXPORT_SECRET_KEY) {
    die('Access denied. Please provide a valid key parameter.');
}

// Get export format
$format = isset($_GET['format']) ? $_GET['format'] : 'csv';
$status = isset($_GET['status']) ? $_GET['status'] : 'all';

// Connect to database
global $wpdb;
$members_table = $wpdb->prefix . 'wpmm_members';
$meta_table = $wpdb->prefix . 'wpmm_member_meta';

// Build query
$where = array();
if ($status !== 'all') {
    $where[] = $wpdb->prepare('status = %s', $status);
} else {
    $where[] = "status != 'deleted'";
}

$where_clause = implode(' AND ', $where);

// Get members
$members = $wpdb->get_results(
    "SELECT * FROM {$members_table} WHERE {$where_clause} ORDER BY registration_date DESC",
    ARRAY_A
);

// Get meta for each member
foreach ($members as &$member) {
    $meta = $wpdb->get_results($wpdb->prepare(
        "SELECT meta_key, meta_value FROM {$meta_table} WHERE member_id = %s",
        $member['member_id']
    ), ARRAY_A);
    
    $member['meta'] = array();
    foreach ($meta as $m) {
        $member['meta'][$m['meta_key']] = $m['meta_value'];
    }
    
    // Remove sensitive data
    unset($member['password_hash']);
}

// Export based on format
switch ($format) {
    case 'json':
        header('Content-Type: application/json');
        header('Content-Disposition: attachment; filename="members-export.json"');
        echo json_encode($members, JSON_PRETTY_PRINT);
        break;
        
    case 'csv':
    default:
        header('Content-Type: text/csv; charset=utf-8');
        header('Content-Disposition: attachment; filename="members-export.csv"');
        
        $output = fopen('php://output', 'w');
        
        // Headers
        $headers = array('member_id', 'username', 'email', 'first_name', 'last_name', 
                        'display_name', 'membership_level', 'status', 'email_verified',
                        'registration_date', 'last_login');
        fputcsv($output, $headers);
        
        // Data
        foreach ($members as $member) {
            $row = array();
            foreach ($headers as $field) {
                $row[] = $member[$field] ?? '';
            }
            fputcsv($output, $row);
        }
        
        fclose($output);
        break;
}
```

## 3. WP-CLI COMMAND FOR EXPORT

### Add to includes/class-wp-cli-commands.php

```php
<?php
/**
 * WP-CLI Commands for Member Export
 * 
 * Usage:
 * wp member export --format=csv
 * wp member export --format=json --status=active
 * wp member export --format=csv --output=/path/to/file.csv
 */

if (!defined('WP_CLI') || !WP_CLI) {
    return;
}

class WPMM_CLI_Commands {
    
    /**
     * Export members to file
     *
     * ## OPTIONS
     *
     * [--format=<format>]
     * : Export format: csv, json, xml
     * Default: csv
     *
     * [--status=<status>]
     * : Member status: all, active, pending, suspended, deleted
     * Default: all
     *
     * [--output=<file>]
     * : Output file path. If not provided, outputs to STDOUT
     *
     * [--fields=<fields>]
     * : Comma-separated list of fields to export
     * Default: member_id,username,email,first_name,last_name,membership_level,status,registration_date
     *
     * ## EXAMPLES
     *
     *     wp member export --format=csv --status=active
     *     wp member export --format=json --output=/tmp/members.json
     */
    public function export($args, $assoc_args) {
        global $wpdb;
        
        $format = $assoc_args['format'] ?? 'csv';
        $status = $assoc_args['status'] ?? 'all';
        $output_file = $assoc_args['output'] ?? null;
        $fields = isset($assoc_args['fields']) ? explode(',', $assoc_args['fields']) : 
                 array('member_id', 'username', 'email', 'first_name', 'last_name', 
                       'membership_level', 'status', 'registration_date');
        
        // Build query
        $where = array();
        if ($status !== 'all') {
            $where[] = $wpdb->prepare('status = %s', $status);
        } else {
            $where[] = "status != 'deleted'";
        }
        
        $where_clause = implode(' AND ', $where);
        
        // Get members
        WP_CLI::log('Fetching members...');
        
        $members = $wpdb->get_results(
            "SELECT * FROM {$wpdb->prefix}wpmm_members WHERE {$where_clause} ORDER BY registration_date DESC",
            ARRAY_A
        );
        
        $total = count($members);
        WP_CLI::log("Found {$total} members");
        
        if ($total === 0) {
            WP_CLI::warning('No members found matching criteria.');
            return;
        }
        
        // Prepare data
        $export_data = array();
        foreach ($members as $member) {
            $row = array();
            foreach ($fields as $field) {
                $row[$field] = $member[$field] ?? '';
            }
            $export_data[] = $row;
        }
        
        // Generate output
        switch ($format) {
            case 'json':
                $content = json_encode($export_data, JSON_PRETTY_PRINT);
                $extension = 'json';
                break;
                
            case 'xml':
                $xml = new SimpleXMLElement('<?xml version="1.0"?><members></members>');
                foreach ($export_data as $data) {
                    $member_xml = $xml->addChild('member');
                    foreach ($data as $key => $value) {
                        $member_xml->addChild($key, htmlspecialchars($value));
                    }
                }
                $content = $xml->asXML();
                $extension = 'xml';
                break;
                
            case 'csv':
            default:
                $output = fopen('php://temp', 'r+');
                fputcsv($output, array_keys($export_data[0]));
                foreach ($export_data as $row) {
                    fputcsv($output, $row);
                }
                rewind($output);
                $content = stream_get_contents($output);
                fclose($output);
                $extension = 'csv';
                break;
        }
        
        // Save to file or output
        if ($output_file) {
            $output_file = str_replace('.csv', '.' . $extension, $output_file);
            file_put_contents($output_file, $content);
            WP_CLI::success("Exported {$total} members to: {$output_file}");
        } else {
            WP_CLI::log($content);
        }
    }
    
    /**
     * Count members
     *
     * ## OPTIONS
     *
     * [--status=<status>]
     * : Member status to count
     *
     * ## EXAMPLES
     *
     *     wp member count
     *     wp member count --status=active
     */
    public function count($args, $assoc_args) {
        global $wpdb;
        
        $status = $assoc_args['status'] ?? 'all';
        
        $where = array();
        if ($status !== 'all') {
            $where[] = $wpdb->prepare('status = %s', $status);
        } else {
            $where[] = "status != 'deleted'";
        }
        
        $where_clause = implode(' AND ', $where);
        
        $count = $wpdb->get_var(
            "SELECT COUNT(*) FROM {$wpdb->prefix}wpmm_members WHERE {$where_clause}"
        );
        
        $by_level = $wpdb->get_results(
            "SELECT membership_level, COUNT(*) as count 
             FROM {$wpdb->prefix}wpmm_members 
             WHERE {$where_clause} 
             GROUP BY membership_level",
            ARRAY_A
        );
        
        WP_CLI::log("Total members: {$count}");
        WP_CLI::log("\nBy membership level:");
        
        foreach ($by_level as $level) {
            WP_CLI::log("  {$level['membership_level']}: {$level['count']}");
        }
    }
}

WP_CLI::add_command('member', 'WPMM_CLI_Commands');
```

## 4. PHP MYADMIN EXPORT QUERIES

### sql-export-queries.sql

```sql
-- ============================================
-- SQL Queries for Manual Export via phpMyAdmin
-- ============================================

-- 1. EXPORT ALL ACTIVE MEMBERS
SELECT 
    member_id,
    username,
    email,
    first_name,
    last_name,
    display_name,
    membership_level,
    status,
    email_verified,
    registration_date,
    last_login,
    last_activity
FROM wp_wpmm_members
WHERE status = 'active'
ORDER BY registration_date DESC;

-- 2. EXPORT WITH CUSTOM META FIELDS
SELECT 
    m.member_id,
    m.username,
    m.email,
    m.first_name,
    m.last_name,
    m.membership_level,
    m.status,
    m.registration_date,
    MAX(CASE WHEN meta.meta_key = 'registration_ip' THEN meta.meta_value END) as ip_address,
    MAX(CASE WHEN meta.meta_key = 'phone' THEN meta.meta_value END) as phone,
    MAX(CASE WHEN meta.meta_key = 'company' THEN meta.meta_value END) as company
FROM wp_wpmm_members m
LEFT JOIN wp_wpmm_member_meta meta ON m.member_id = meta.member_id
WHERE m.status = 'active'
GROUP BY m.member_id
ORDER BY m.registration_date DESC;

-- 3. EXPORT MEMBERS BY DATE RANGE
SELECT 
    member_id,
    username,
    email,
    first_name,
    last_name,
    membership_level,
    status,
    registration_date
FROM wp_wpmm_members
WHERE registration_date BETWEEN '2024-01-01' AND '2024-12-31'
ORDER BY registration_date DESC;

-- 4. EXPORT WITH RELATED PLUGIN DATA (Example: Payments)
SELECT 
    m.member_id,
    m.username,
    m.email,
    m.first_name,
    m.last_name,
    m.membership_level,
    m.registration_date,
    COUNT(p.id) as total_payments,
    SUM(p.amount) as total_spent,
    MAX(p.created_at) as last_payment_date
FROM wp_wpmm_members m
LEFT JOIN wp_payment_transactions p ON m.member_id = p.user_id
WHERE m.status = 'active'
GROUP BY m.member_id
ORDER BY total_spent DESC;

-- 5. EXPORT MEMBER COUNT BY LEVEL
SELECT 
    membership_level,
    COUNT(*) as total,
    SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) as active,
    SUM(CASE WHEN status = 'pending' THEN 1 ELSE 0 END) as pending,
    SUM(CASE WHEN status = 'suspended' THEN 1 ELSE 0 END) as suspended
FROM wp_wpmm_members
WHERE status != 'deleted'
GROUP BY membership_level;

-- 6. EXPORT ALL MEMBER META DATA
SELECT 
    m.member_id,
    m.username,
    m.email,
    meta.meta_key,
    meta.meta_value
FROM wp_wpmm_members m
JOIN wp_wpmm_member_meta meta ON m.member_id = meta.member_id
WHERE m.status = 'active'
ORDER BY m.member_id, meta.meta_key;

-- 7. EXPORT SESSION DATA
SELECT 
    s.session_id,
    s.member_id,
    m.username,
    m.email,
    s.ip_address,
    s.login_time,
    s.last_activity,
    s.expiry_time,
    s.is_active
FROM wp_wpmm_sessions s
JOIN wp_wpmm_members m ON s.member_id = m.member_id
ORDER BY s.login_time DESC;

-- 8. EXPORT MEMBER STATISTICS
SELECT 
    DATE(registration_date) as date,
    COUNT(*) as new_members,
    SUM(CASE WHEN membership_level != 'free' THEN 1 ELSE 0 END) as paid_members
FROM wp_wpmm_members
WHERE status = 'active'
GROUP BY DATE(registration_date)
ORDER BY date DESC;
```

## 5. ONE-CLICK EXPORT BUTTON (Add to Main Plugin)

### Add to wp-member-manager.php

```php
/**
 * Add quick export button to admin bar
 */
add_action('admin_bar_menu', 'wpmm_add_export_button', 100);
function wpmm_add_export_button($admin_bar) {
    if (!current_user_can('manage_options')) {
        return;
    }
    
    $admin_bar->add_menu(array(
        'id'    => 'wpmm-export',
        'parent' => null,
        'group'  => null,
        'title' => 'Export Members',
        'href'  => admin_url('admin.php?page=wpmm-export'),
        'meta'  => array(
            'title' => __('Export Members', 'wp-member-manager'),
        ),
    ));
}

/**
 * Add export link to plugins page
 */
add_filter('plugin_action_links_' . plugin_basename(__FILE__), 'wpmm_add_export_link');
function wpmm_add_export_link($links) {
    $export_link = '<a href="' . admin_url('admin.php?page=wpmm-export') . '">' . 
                   __('Export Members', 'wp-member-manager') . '</a>';
    array_unshift($links, $export_link);
    return $links;
}
```

## 6. QUICK EXPORT SHORTCODE (For Frontend)

### Add shortcode for admin-only frontend export

```php
/**
 * Shortcode: Quick Export Button (Admin Only)
 * Usage: [wpmm_export_button]
 */
add_shortcode('wpmm_export_button', 'wpmm_export_button_shortcode');
function wpmm_export_button_shortcode() {
    if (!current_user_can('manage_options')) {
        return '';
    }
    
    ob_start();
    ?>
    <div class="wpmm-export-shortcode">
        <h3>Quick Export</h3>
        
        <form method="post" action="<?php echo admin_url('admin-ajax.php'); ?>">
            <input type="hidden" name="action" value="wpmm_quick_export">
            <?php wp_nonce_field('wpmm_quick_export'); ?>
            
            <select name="format">
                <option value="csv">CSV</option>
                <option value="json">JSON</option>
            </select>
            
            <select name="status">
                <option value="all">All Members</option>
                <option value="active">Active Only</option>
            </select>
            
            <button type="submit" class="button">Export Now</button>
        </form>
    </div>
    <?php
    return ob_get_clean();
}

// Handle quick export AJAX
add_action('wp_ajax_wpmm_quick_export', 'wpmm_handle_quick_export');
function wpmm_handle_quick_export() {
    if (!current_user_can('manage_options') || !wp_verify_nonce($_POST['_wpnonce'], 'wpmm_quick_export')) {
        wp_die('Unauthorized');
    }
    
    global $wpdb;
    $format = $_POST['format'] ?? 'csv';
    $status = $_POST['status'] ?? 'all';
    
    $where = ($status !== 'all') ? $wpdb->prepare('WHERE status = %s', $status) : "WHERE status != 'deleted'";
    
    $members = $wpdb->get_results(
        "SELECT member_id, username, email, first_name, last_name, membership_level, status, registration_date 
         FROM {$wpdb->prefix}wpmm_members {$where} 
         ORDER BY registration_date DESC",
        ARRAY_A
    );
    
    if ($format === 'json') {
        header('Content-Type: application/json');
        header('Content-Disposition: attachment; filename="members-export.json"');
        echo json_encode($members, JSON_PRETTY_PRINT);
    } else {
        header('Content-Type: text/csv');
        header('Content-Disposition: attachment; filename="members-export.csv"');
        
        $output = fopen('php://output', 'w');
        fputcsv($output, array_keys($members[0]));
        foreach ($members as $member) {
            fputcsv($output, $member);
        }
        fclose($output);
    }
    
    exit;
}
```

## HOW TO USE THESE EXPORT METHODS:

### **Method 1: Admin Panel Export**
1. Go to WordPress Admin → WP Member Manager → Export Members
2. Select format (CSV, JSON, XML, Excel)
3. Filter by status, membership level, date range
4. Click "Export Members"

### **Method 2: Direct Database Access**
1. Open phpMyAdmin or similar tool
2. Select your WordPress database
3. Run any of the SQL queries provided above
4. Export results to CSV/JSON

### **Method 3: WP-CLI Command**
```bash
# Export all active members as CSV
wp member export --format=csv --status=active

# Export to file
wp member export --format=json --output=/home/user/exports/members.json

# Count members
wp member count --status=active
```

### **Method 4: Direct Script**
1. Upload `export-members-direct.php` to WordPress root
2. Access via browser: `https://yoursite.com/export-members-direct.php?key=YOUR_SECRET_KEY`
3. **DELETE THE FILE IMMEDIATELY AFTER USE**

### **Method 5: Quick Export Shortcode**
Add `[wpmm_export_button]` to any page (visible to admins only)

The export system includes:
- ✅ Multiple formats (CSV, JSON, XML, Excel)
- ✅ Filter by status, membership level, dates
- ✅ Select specific fields
- ✅ Include meta data
- ✅ Export history tracking
- ✅ Preview before export
- ✅ Batch export for large datasets
- ✅ WP-CLI support
- ✅ Direct database queries
