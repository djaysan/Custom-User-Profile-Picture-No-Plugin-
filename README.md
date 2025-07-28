This snippet adds a custom profile picture upload field to the WordPress user profile page. It stores the avatar image as a media attachment linked to the user and overrides the default Gravatar display with the uploaded image site-wide, including in Elementor’s author image widget.

Features
✅ Adds an upload field under “Profile Picture” on user profile edit pages
✅ Saves and manages the image via WordPress media library
✅ Overrides Gravatar with uploaded image everywhere WordPress uses get_avatar()
✅ No third-party plugins required — uses WordPress core functions only
✅ Compatible with Elementor’s dynamic author image widget

How It Works
Adds a file input on user profile pages under the "Profile Picture" section

Handles file upload, stores the image as an attachment, and saves its ID in user meta

Hooks into get_avatar filter to display the uploaded image instead of Gravatar

Works transparently with theme and plugins that use standard WordPress avatar functions

Installation
Add the following PHP code to your theme’s functions.php file or a custom plugin:

```
<?php
// Add upload field under "Profile Picture" on user profile page
function custom_user_avatar_field( $user ) {
    ?>
    <h3><?php esc_html_e( 'Profile Picture', 'custom' ); ?></h3>
    <table class="form-table">
        <tr>
            <th><label for="custom_user_avatar"><?php esc_html_e( 'Upload Image', 'custom' ); ?></label></th>
            <td>
                <?php
                $avatar_id = get_user_meta( $user->ID, 'custom_user_avatar', true );
                if ( $avatar_id ) {
                    echo wp_get_attachment_image( $avatar_id, [96,96], false, [ 'style' => 'border-radius:50%;display:block;margin-bottom:10px;' ] );
                }
                ?>
                <input type="file" name="custom_user_avatar" id="custom_user_avatar" /><br />
                <span class="description"><?php esc_html_e( 'Upload a custom profile picture.', 'custom' ); ?></span>
            </td>
        </tr>
    </table>
    <?php
}
add_action( 'show_user_profile', 'custom_user_avatar_field' );
add_action( 'edit_user_profile', 'custom_user_avatar_field' );

// Handle avatar upload and save attachment ID as user meta
function custom_save_user_avatar( $user_id ) {
    if ( ! current_user_can( 'edit_user', $user_id ) ) {
        return;
    }

    if ( ! empty( $_FILES['custom_user_avatar']['name'] ) ) {
        require_once ABSPATH . 'wp-admin/includes/file.php';
        require_once ABSPATH . 'wp-admin/includes/media.php';
        require_once ABSPATH . 'wp-admin/includes/image.php';

        $attachment_id = media_handle_upload( 'custom_user_avatar', 0 );
        if ( is_wp_error( $attachment_id ) ) {
            // Handle upload error if needed
            return;
        }

        update_user_meta( $user_id, 'custom_user_avatar', $attachment_id );
    }
}
add_action( 'personal_options_update', 'custom_save_user_avatar' );
add_action( 'edit_user_profile_update', 'custom_save_user_avatar' );

// Override avatar output to use uploaded image if available
function custom_get_avatar( $avatar, $id_or_email, $size, $default, $alt ) {
    $user_id = 0;

    if ( is_numeric( $id_or_email ) ) {
        $user_id = (int) $id_or_email;
    } elseif ( is_object( $id_or_email ) && ! empty( $id_or_email->user_id ) ) {
        $user_id = (int) $id_or_email->user_id;
    } else {
        $user = get_user_by( 'email', $id_or_email );
        if ( $user ) {
            $user_id = $user->ID;
        }
    }

    if ( $user_id ) {
        $avatar_id = get_user_meta( $user_id, 'custom_user_avatar', true );
        if ( $avatar_id ) {
            $avatar = wp_get_attachment_image( $avatar_id, [ $size, $size ], false, [ 'alt' => $alt, 'class' => 'avatar avatar-custom' ] );
        }
    }

    return $avatar;
}
add_filter( 'get_avatar', 'custom_get_avatar', 10, 5 );

```
Notes
Uploaded images are stored in the Media Library.

The avatar replaces Gravatar everywhere WordPress uses get_avatar().

Compatible with Elementor’s dynamic author image widget.

No external plugin dependencies.

License
Feel free to use and modify as needed.

