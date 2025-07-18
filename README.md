<?php
/**
 * Plugin Name: WP Notification System
 * Plugin URI: https://github.com/arclub99/wp-notification-system
 * Description: Custom frontend popup notifications with admin post control.
 * Version: 1.1.2
 * Author: RashidVerse
 * Author URI: https://rashidverse.com/
 * License: GPL2
 * Text Domain: wp-notification-system
 */

// Register Custom Post Type
add_action('init', function() {
    register_post_type('site_notification', [
        'labels' => [
            'name' => __('Notifications'),
            'singular_name' => __('Notification')
        ],
        'public' => false,
        'show_ui' => true,
        'show_in_menu' => true,
        'menu_position' => 25,
        'menu_icon' => 'dashicons-megaphone',
        'supports' => ['title', 'editor', 'author']
    ]);
});

// Settings UI
add_action('admin_menu', function() {
    add_submenu_page('edit.php?post_type=site_notification', 'Notification Settings', 'Settings', 'manage_options', 'notification-settings', function() {
        ?>
        <div class="wrap">
            <h1>Notification Settings</h1>
            <form method="post" action="options.php">
                <?php
                    settings_fields('notification_settings');
                    do_settings_sections('notification-settings');
                    submit_button();
                ?>
            </form>
        </div>
        <?php
    });
});

add_action('admin_init', function() {
    register_setting('notification_settings', 'notification_icon_color');
    register_setting('notification_settings', 'notification_icon_size');
    register_setting('notification_settings', 'notification_icon_position');
    register_setting('notification_settings', 'notification_icon_custom_left');
    register_setting('notification_settings', 'notification_icon_custom_bottom');

    add_settings_section('notification_settings_section', '', null, 'notification-settings');

    add_settings_field('notification_icon_color', 'Icon Color', function() {
        $val = esc_attr(get_option('notification_icon_color', '#ff0000'));
        echo "<input type='color' name='notification_icon_color' value='{$val}'>";
    }, 'notification-settings', 'notification_settings_section');

    add_settings_field('notification_icon_size', 'Icon Size (px)', function() {
        $val = esc_attr(get_option('notification_icon_size', 48));
        echo "<input type='number' name='notification_icon_size' value='{$val}'>";
    }, 'notification-settings', 'notification_settings_section');

    add_settings_field('notification_icon_position', 'Icon Position', function() {
        $val = esc_attr(get_option('notification_icon_position', 'bottom-left'));
        echo "<select name='notification_icon_position'>
                <option value='top-left'" . selected($val, 'top-left', false) . ">Top Left</option>
                <option value='top-right'" . selected($val, 'top-right', false) . ">Top Right</option>
                <option value='bottom-left'" . selected($val, 'bottom-left', false) . ">Bottom Left</option>
                <option value='bottom-right'" . selected($val, 'bottom-right', false) . ">Bottom Right</option>
                <option value='custom'" . selected($val, 'custom', false) . ">Custom</option>
              </select>";
    }, 'notification-settings', 'notification_settings_section');

    add_settings_field('notification_icon_custom_left', 'Custom Left (px)', function() {
        $val = esc_attr(get_option('notification_icon_custom_left', 200));
        echo "<input type='number' name='notification_icon_custom_left' value='{$val}'>";
    }, 'notification-settings', 'notification_settings_section');

    add_settings_field('notification_icon_custom_bottom', 'Custom Bottom (px)', function() {
        $val = esc_attr(get_option('notification_icon_custom_bottom', 40));
        echo "<input type='number' name='notification_icon_custom_bottom' value='{$val}'>";
    }, 'notification-settings', 'notification_settings_section');
});

// Frontend: Inject notification icon and popup
add_action('wp_footer', function() {
    echo do_shortcode('[notification_popup]');
});

add_shortcode('notification_popup', function() {
    $args = [
        'post_type' => 'site_notification',
        'post_status' => 'publish',
        'posts_per_page' => 10,
        'orderby' => 'date',
        'order' => 'DESC',
    ];
    $query = new WP_Query($args);
    $default_count = $query->found_posts;
    $count = esc_attr(get_option('notification_icon_custom_count', '')) ?: $default_count;
    $color = esc_attr(get_option('notification_icon_color', '#ff0000'));
    $size = intval(get_option('notification_icon_size', 48));
    $position = esc_attr(get_option('notification_icon_position', 'bottom-left'));
    $left = intval(get_option('notification_icon_custom_left', 200));
    $bottom = intval(get_option('notification_icon_custom_bottom', 40));
    $svg = get_option('notification_icon_svg');

    $position_css = '';
    switch ($position) {
        case 'top-left': $position_css = "top:20px;left:20px;"; break;
        case 'top-right': $position_css = "top:20px;right:20px;"; break;
        case 'bottom-right': $position_css = "bottom:20px;right:20px;"; break;
        case 'custom': $position_css = "bottom:{$bottom}px;left:{$left}px;"; break;
        default: $position_css = "bottom:20px;left:20px;";
    }

    ob_start();
    ?>
    <style>
    .notif-icon {
        position: fixed;
        <?= $position_css ?>
        background-color: <?= $color ?>;
        border-radius: 50%;
        width: <?= $size ?>px;
        height: <?= $size ?>px;
        z-index: 9999;
        cursor: pointer;
        display: flex;
        justify-content: center;
        align-items: center;
        transition: all 0.3s;
    }
    .notif-icon svg {
        width: 60%;
        height: 60%;
        fill: white;
    }
    .notif-count {
        position: absolute;
        top: -6px;
        right: -6px;
        background-color: white;
        color: <?= $color ?>;
        border-radius: 999px;
        font-size: 11px;
        padding: 2px 6px;
        font-weight: bold;
        line-height: 1;
        border: 1px solid <?= $color ?>;
    }
    .notif-overlay {
        position: fixed;
        top: 0; left: 0; width: 100%; height: 100%;
        background: rgba(0,0,0,0.6);
        display: none;
        z-index: 9998;
    }
    .notif-popup {
        position: fixed;
        top: 50%; left: 50%;
        transform: translate(-50%, -50%) scale(0.9);
        width: 90%; max-width: 780px; max-height: 90vh;
        background: white; border: 3px solid <?= $color ?>;
        border-radius: 8px;
        z-index: 9999;
        display: none;
        overflow-y: auto;
        animation: macOpen 0.35s ease-out forwards;
    }
    .notif-popup-header {
        background-color: <?= $color ?>;
        color: white;
        padding: 10px 15px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        font-weight: bold;
    }
    .notif-popup-content { padding: 20px; font-size: 15px; line-height: 1.5; }
    .notif-view-all { text-align: center; margin-top: 20px; }
    .notif-view-all a {
        background: <?= $color ?>;
        color: white;
        padding: 8px 16px;
        text-decoration: none;
        border-radius: 5px;
    }
    @keyframes shake {
        0%,100%{transform:rotate(0);} 25%{transform:rotate(15deg);} 50%{transform:rotate(-15deg);} 75%{transform:rotate(10deg);}
    }
    @keyframes macOpen {
        0%{opacity:0;transform:translate(-50%,-50%) scale(0.7);}
        100%{opacity:1;transform:translate(-50%,-50%) scale(1);}
    }
    </style>
    <div class="notif-icon" onclick="openNotificationPopup()">
        <?= $svg ? $svg : '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M12 2a7 7 0 00-7 7v5H4a1 1 0 000 2h16a1 1 0 000-2h-1V9a7 7 0 00-7-7zM9 21a3 3 0 006 0H9z"/></svg>' ?>
        <div class="notif-count" id="notifCount"><?= $count ?></div>
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
                <div class="notif-full-content" style="display:none;"> <?php the_content(); ?> </div>
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
        const bell = document.querySelector('.notif-icon svg');
        if (bell) {
            bell.style.animation = 'shake 0.5s';
            setTimeout(() => { bell.style.animation = ''; }, 1000);
        }
    }, 3000);

    function openNotificationPopup() {
        document.getElementById('notifPopup').style.display = 'block';
        document.getElementById('notifOverlay').style.display = 'block';
        const countElem = document.getElementById('notifCount');
        if (countElem) countElem.style.display = 'none';
    }
    function closeNotificationPopup() {
        document.getElementById('notifPopup').style.display = 'none';
        document.getElementById('notifOverlay').style.display = 'none';
    }
    function toggleNotifContent(btn) {
        const full = btn.previousElementSibling;
        if (full.style.display === 'block') {
            full.style.display = 'none';
            btn.textContent = 'See more';
        } else {
            full.style.display = 'block';
            btn.textContent = 'See less';
        }
    }
    </script>
    <?php
    return ob_get_clean();
});

// Virtual Page: All Notifications
add_action('init', function() {
    add_rewrite_rule('^all-notifications/?$', 'index.php?all_notifications_page=1', 'top');
    add_rewrite_tag('%all_notifications_page%', '1');
});

add_action('template_redirect', function() {
    if (get_query_var('all_notifications_page')) {
        status_header(200);
        get_header();

        echo '<main class="site-main" style="max-width:800px;margin:40px auto;padding:0 20px;">';
        echo '<h1>All Notifications</h1>';

        $args = [
            'post_type' => 'site_notification',
            'post_status' => 'publish',
            'posts_per_page' => -1,
            'orderby' => 'date',
            'order' => 'DESC',
        ];
        $query = new WP_Query($args);
        if ($query->have_posts()) {
            while ($query->have_posts()) {
                $query->the_post();
                echo '<article style="margin-bottom:30px;">
                        <h2>' . get_the_title() . '</h2>
                        <small>' . get_the_date() . ' | By ' . get_the_author() . '</small>
                        <div>' . apply_filters('the_content', get_the_content()) . '</div>
                      </article>';
            }
            wp_reset_postdata();
        } else {
            echo '<p>No notifications found.</p>';
        }

        echo '</main>';
        get_footer();
        exit;
    }
});
