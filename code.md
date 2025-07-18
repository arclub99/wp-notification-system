# wp-notification-system
wp-notification-system

<?php
/**
 * Plugin Name: WP Notification System
 * Plugin URI: https://yourdomain.com/wp-notification-system
 * Description: Custom frontend popup notifications with admin post control.
 * Version: 1.0.0
 * Author: RashidVerse
 * Author URI: https://rashidverse.com/
 * License: GPL2
 * Text Domain: wp-notification-system
 */


// Plugin functionality for custom notifications in WordPress

// 1. Register Custom Post Type for Notifications
add_action('init', function() {
    register_post_type('site_notification', [
        'labels' => [
            'name' => __('Notifications'),
            'singular_name' => __('Notification'),
            'add_new' => __('Create New Notification'),
            'add_new_item' => __('Add New Notification'),
            'edit_item' => __('Edit Notification'),
            'new_item' => __('New Notification'),
            'view_item' => __('View Notification'),
            'search_items' => __('Search Notifications'),
        ],
        'public' => false,
        'show_ui' => true,
        'show_in_menu' => true,
        'menu_position' => 25,
        'menu_icon' => 'dashicons-megaphone',
        'supports' => ['title', 'editor', 'author'],
    ]);
});

// 2. Shortcode to show notifications popup with icon
add_shortcode('notification_popup', function() {
    $args = [
        'post_type' => 'site_notification',
        'post_status' => 'publish',
        'posts_per_page' => 10,
        'orderby' => 'date',
        'order' => 'DESC',
    ];
    $query = new WP_Query($args);
    $count = $query->found_posts;

    ob_start();
    ?>
    <style>
    .notif-full-content { display: none; margin-top: 10px; }
    .notif-readmore, .notif-readless {
        color: #ff0000;
        cursor: pointer;
        font-size: 14px;
        margin-top: 6px;
        display: inline-block;
    }
    .notif-icon {
        position: fixed;
        bottom: 20px;
        left: 20px;
        background-color: #ff0000;
        border-radius: 50%;
        padding: 12px;
        z-index: 1001;
        cursor: pointer;
    }
    .notif-icon svg {
        width: 24px;
        height: 24px;
        fill: white;
        display: block;
    }
    .notif-count {
    position: absolute;
    top: -6px;
    right: -6px;
    background-color: white;
    color: #ff0000;
    border-radius: 999px;
    font-size: 11px;
    padding: 2px 6px;
    font-weight: bold;
    line-height: 1;
    min-width: 18px;
    text-align: center;
    border: 1px solid red;
}

    .notif-overlay {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: rgba(0, 0, 0, 0.6);
        display: none;
        z-index: 1000;
    }
    .notif-popup {
        position: fixed;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%) scale(0.95);
        width: 90%;
        max-width: 780px;
        max-height: 90vh;
        overflow-y: auto;
        background: white;
        border: 3px solid #ff0000;
        border-radius: 8px;
        z-index: 1002;
        display: none;
        animation: macOpen 0.35s ease-out forwards;
    }
    .notif-popup-header {
        background-color: #ff0000;
        color: white;
        padding: 10px 15px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        font-weight: bold;
    }
    .notif-popup-content {
        padding: 20px;
        font-size: 15px;
        line-height: 1.5;
    }
    .notif-view-all {
        text-align: center;
        margin-top: 20px;
    }
    .notif-view-all a {
        background: #ff0000;
        color: white;
        padding: 8px 16px;
        text-decoration: none;
        border-radius: 5px;
    }
    @keyframes shake {
        0%, 100% { transform: rotate(0); }
        25% { transform: rotate(15deg); }
        50% { transform: rotate(-15deg); }
        75% { transform: rotate(10deg); }
    }
    @keyframes macOpen {
        0% { transform: translate(-50%, -50%) scale(0.7); opacity: 0; }
        100% { transform: translate(-50%, -50%) scale(1); opacity: 1; }
    }
    </style>
    <div class="notif-icon" onclick="openNotificationPopup()">
        <svg id="notifBell" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M12 24a2.5 2.5 0 0 0 2.45-2h-4.9A2.5 2.5 0 0 0 12 24zm6.36-6V11c0-3.52-2.61-6.43-6-6.92V3a1 1 0 0 0-2 0v1.08c-3.39.49-6 3.4-6 6.92v7L2 20v1h20v-1l-3.64-2z"/></svg>
        <div class="notif-count" id="notifCount"><?php echo $count; ?></div>
    </div>
    <div class="notif-overlay" id="notifOverlay" onclick="closeNotificationPopup()"></div>
    <div class="notif-popup" id="notifPopup">
        <div class="notif-popup-header">
            <strong>Latest Updates</strong>
            <span onclick="closeNotificationPopup()" style="cursor:pointer">&times;</span>
        </div>
        <div class="notif-popup-content">
        <?php if ($query->have_posts()): while ($query->have_posts()): $query->the_post(); ?>
            <div style="margin-bottom:15px; border-bottom:1px solid #ccc; padding-bottom:10px;">
                <strong><?php the_title(); ?></strong><br>
                <small><?php echo get_the_date(); ?> | By <?php the_author(); ?></small>
                <p class="notif-excerpt"><?php echo wp_trim_words(get_the_content(), 20); ?></p>
                <div class="notif-full-content">
                    <?php the_content(); ?>
                </div>
                <span class="notif-readmore" onclick="toggleNotifContent(this)">See more</span>
            </div>
        <?php endwhile; wp_reset_postdata(); else: ?>
            <p>No notifications available.</p>
        <?php endif; ?>
        <div class="notif-view-all">
            <a href="<?php echo site_url('/all-notifications'); ?>">See all updates</a>
        </div>
        </div>
    </div>
    <script>
    setInterval(() => {
        const bell = document.getElementById('notifBell');
        bell.style.animation = 'shake 0.5s';
        setTimeout(() => {
            bell.style.animation = '';
        }, 500);
    }, 10000);

    function openNotificationPopup() {
        const popup = document.getElementById('notifPopup');
        popup.style.display = 'block';
        popup.style.animation = 'macOpen 0.35s ease-out';
        document.getElementById('notifOverlay').style.display = 'block';
        const countElem = document.getElementById('notifCount');
        if (countElem) countElem.style.display = 'none';
    }
    function closeNotificationPopup() {
        document.getElementById('notifPopup').style.display = 'none';
        document.getElementById('notifOverlay').style.display = 'none';
    }
    function toggleNotifContent(btn) {
        const fullContent = btn.previousElementSibling;
        if (fullContent.style.display === 'block') {
            fullContent.style.display = 'none';
            btn.textContent = 'See more';
        } else {
            fullContent.style.display = 'block';
            btn.textContent = 'See less';
            fullContent.scrollIntoView({behavior: 'smooth'});
        }
    }
    </script>
    <?php
    return ob_get_clean();
});

// 3. Register a page template to show all notifications
add_action('init', function() {
    add_rewrite_rule('^all-notifications/?$', 'index.php?all_notifications_page=1', 'top');
});

add_filter('query_vars', function($vars) {
    $vars[] = 'all_notifications_page';
    return $vars;
});

add_action('template_redirect', function() {
    if (get_query_var('all_notifications_page')) {
        get_header();
        echo '<div style="max-width:800px; margin:40px auto; padding:20px">';
        echo '<h2>All Updates</h2>';
        $args = [
            'post_type' => 'site_notification',
            'post_status' => 'publish',
            'orderby' => 'date',
            'order' => 'DESC',
            'posts_per_page' => -1,
        ];
        $query = new WP_Query($args);
        if ($query->have_posts()):
            while ($query->have_posts()): $query->the_post();
                echo '<div style="margin-bottom:20px; padding-bottom:15px; border-bottom:1px solid #ddd">';
                echo '<h3>' . get_the_title() . '</h3>';
                echo '<small>' . get_the_date() . ' | By ' . get_the_author() . '</small>';
                echo '<p>' . get_the_content() . '</p>';
                echo '</div>';
            endwhile;
            wp_reset_postdata();
        else:
            echo '<p>No notifications yet.</p>';
        endif;
        echo '</div>';
        get_footer();
        exit;
    }
});
