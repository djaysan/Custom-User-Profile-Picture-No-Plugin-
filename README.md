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
/**
 * Custom User Profile Avatar (Replaces Gravatar + Works with Elementor)
 */

// === 1. Add custom avatar field ===
add_action( 'show_user_profile', 'custom_user_avatar_field' );
add_action( 'edit_user_profile', 'custom_user_avatar_field' );
function custom_user_avatar_field( $user ) {
    $avatar_id = get_user_meta( $user->ID, 'custom_user_avatar_id', true );
    $avatar_url = $avatar_id ? wp_get_attachment_image_url( $avatar_id, 'thumbnail' ) : '';
    ?>
    <div id="custom-avatar-field" style="display:none;">
        <h3>Profile Picture</h3>
        <table class="form-table">
            <tr>
                <th><label for="custom_user_avatar">Profile Image</label></th>
                <td>
                    <img id="custom_user_avatar_preview" src="<?php echo esc_url( $avatar_url ); ?>" style="height:70px;width:70px;border-radius:50%;background:#f0f0f0;display:block;margin-bottom:10px;">
                    <input type="hidden" name="custom_user_avatar_id" id="custom_user_avatar" value="<?php echo esc_attr( $avatar_id ); ?>">
                    <button type="button" class="button" id="custom_user_avatar_button">Upload / Select Image</button>
                    <button type="button" class="button" id="custom_user_avatar_remove" style="margin-left:10px;">Remove</button>
                    <p class="description">Upload or select an avatar for this user.</p>
                </td>
            </tr>
        </table>
    </div>
    <?php
}

// === 2. Save avatar field & store URL for Elementor ===
add_action( 'personal_options_update', 'custom_user_avatar_save' );
add_action( 'edit_user_profile_update', 'custom_user_avatar_save' );
function custom_user_avatar_save( $user_id ) {
    if ( ! current_user_can( 'edit_user', $user_id ) ) return;
    $avatar_id = intval( $_POST['custom_user_avatar_id'] );
    update_user_meta( $user_id, 'custom_user_avatar_id', $avatar_id );
    $url = $avatar_id ? wp_get_attachment_image_url( $avatar_id, 'thumbnail' ) : '';
    update_user_meta( $user_id, 'profile_picture', $url ); // Elementor & others use this
}

// === 3. Enqueue scripts & reposition field ===
add_action( 'admin_enqueue_scripts', function( $hook ) {
    if ( in_array( $hook, ['profile.php', 'user-edit.php'] ) ) {
        wp_enqueue_media();
        $js = <<<JS
jQuery(document).ready(function($){
    // Move our field under Biographical Info
    $('#description').closest('tr').after( $('#custom-avatar-field').show().find('table tr') );
    // Hide the default gravatar section
    $('h2:contains("Profile Picture")').closest('table').hide();
    // Media uploader
    var frame;
    $('#custom_user_avatar_button').on('click', function(e){
        e.preventDefault();
        if (frame) { frame.open(); return; }
        frame = wp.media({ title: 'Select or Upload Avatar', button: { text: 'Use this image' }, multiple: false });
        frame.on('select', function(){
            var attachment = frame.state().get('selection').first().toJSON();
            $('#custom_user_avatar').val(attachment.id);
            $('#custom_user_avatar_preview').attr('src', attachment.sizes.thumbnail.url);
        });
        frame.open();
    });
    $('#custom_user_avatar_remove').on('click', function(){
        $('#custom_user_avatar').val('');
        $('#custom_user_avatar_preview').attr('src', '');
    });
});
JS;
        wp_add_inline_script( 'jquery-core', $js );
    }
});

// === 4. Override get_avatar() globally ===
add_filter( 'get_avatar', function( $avatar, $id_or_email, $size, $default, $alt ) {
    $user = false;
    if ( is_numeric( $id_or_email ) ) {
        $user = get_user_by( 'id', absint( $id_or_email ) );
    } elseif ( is_object( $id_or_email ) && ! empty( $id_or_email->user_id ) ) {
        $user = get_user_by( 'id', absint( $id_or_email->user_id ) );
    } elseif ( is_email( $id_or_email ) ) {
        $user = get_user_by( 'email', $id_or_email );
    }
    if ( $user ) {
        $avatar_id = get_user_meta( $user->ID, 'custom_user_avatar_id', true );
        if ( $avatar_id ) {
            return wp_get_attachment_image( $avatar_id, [$size,$size], false, [
                'class' => 'custom-user-avatar',
                'alt'   => esc_attr( $alt ),
                'style' => "border-radius:50%;width:{$size}px;height:{$size}px;"
            ]);
        }
    }
    return $avatar;
}, 10, 5 );

// === 5. Make Elementor & others use our avatar URL when fetching meta ===
add_filter( 'get_the_author_meta', function( $value, $field, $user_id ) {
    if ( in_array( $field, ['profile_picture', 'user_avatar', 'avatar'] ) ) {
        $avatar_id = get_user_meta( $user_id, 'custom_user_avatar_id', true );
        if ( $avatar_id ) {
            return wp_get_attachment_image_url( $avatar_id, 'thumbnail' );
        }
    }
    return $value;
}, 10, 3 );
// === 6. Override get_avatar_url() so Elementor uses our custom avatar URL ===
add_filter( 'get_avatar_url', function( $url, $id_or_email, $args ) {
    $user = false;
    if ( is_numeric( $id_or_email ) ) {
        $user = get_user_by( 'id', absint( $id_or_email ) );
    } elseif ( is_object( $id_or_email ) && ! empty( $id_or_email->user_id ) ) {
        $user = get_user_by( 'id', absint( $id_or_email->user_id ) );
    } elseif ( is_email( $id_or_email ) ) {
        $user = get_user_by( 'email', $id_or_email );
    }
    if ( $user ) {
        $avatar_id = get_user_meta( $user->ID, 'custom_user_avatar_id', true );
        if ( $avatar_id ) {
            return wp_get_attachment_image_url( $avatar_id, 'thumbnail' );
        }
    }
    return $url;
}, 10, 3 );

```
Notes
Uploaded images are stored in the Media Library.

The avatar replaces Gravatar everywhere WordPress uses get_avatar().

Compatible with Elementor’s dynamic author image widget.

No external plugin dependencies.

License
Feel free to use and modify as needed.

