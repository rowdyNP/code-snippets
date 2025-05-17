## Replace a link in the Yoast SEO Breadcrumb trail
```
add_filter( 'wpseo_breadcrumb_output', 'np_wpseo_breadcrumb_output' );
function np_wpseo_breadcrumb_output( $output ){
  if( is_singular() ){
    $from = '<a href="https://www.domain.com/category/food/">Recipes</a>'; // EDIT this to your needs
    $to = '<a href="https://www.domain.com/recipe-index/">Recipes</a>'; // EDIT this to your needs
    $output = str_replace( $from, $to, $output );
  }
return $output;
}
```
## Display Errors even when WordPress doesn't
Courtesy of Zack and Justin at [BigScoots](https://www.bigscoots.com/). Add at the top of `index.php`.
```
ini_set('error_reporting', E_ERROR);
register_shutdown_function("fatal_handler");
function fatal_handler() {
$error = error_get_last();
echo("<pre>");
print_r($error);
}
```
## Add a handy "Clear CF Cache" link in the top admin menu, to do a full purge for a zone.

When clicked, it will make an AJAX request to clear the cache, and show a success/fail message in its place. 5 Seconds later it restored to the original label.

Update the code with a Cloudflare API Token and the Zone. There is no nonce check on this, so use a unique API Token that has permission only to purge the cache:

`Zone > Cache Purge > Purge`

`Include > Specific Zone > (domain)`

This code should not be used for a site on our CF Enterprise, only for a site proxied in its own domain.
```
// add a "Clear CF Cache" item
add_action( 'admin_bar_menu', function( $wp_admin_bar ) {
    if ( ! current_user_can( 'manage_options' ) ) return;
    $wp_admin_bar->add_node( [
        'id'    => 'clear_cf_cache',
        'title' => 'Clear CF Cache',
        'href'  => '#',
        'meta'  => [ 'title' => 'Purge Cloudflare Cache' ],
    ] );
}, 100 );

// enqueue inline script to hijack the click and fire an AJAX request
add_action( 'admin_enqueue_scripts', function() {
 wp_add_inline_script( 'jquery-core', "
    jQuery(function($){
        var \$link = $('#wp-admin-bar-clear_cf_cache a');
        \$link.on('click', function(e){
            e.preventDefault();
            var \$self = $(this);
            \$self.text('Clearing...');
            $.post( ajaxurl, { action: 'purge_cf_cache_ajax' } )
              .done(function(res){
                  \$self.text( res.success ? 'Cache Cleared!' : 'Error' );
                  setTimeout(function(){
                      \$self.text('Clear CF Cache');
                  }, 5000);
              })
              .fail(function(){
                  \$self.text('Error');
                  setTimeout(function(){
                      \$self.text('Clear CF Cache');
                  }, 5000);
              });
        });
    });
" );

});

// handle the AJAX request
add_action( 'wp_ajax_purge_cf_cache_ajax', function() {
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_send_json_error( 'Forbidden', 403 );
    }

    $api_key = 'CLOUDFLARE_API_TOKEN';
    $zone_id = 'CLOUDFLARE_ZONE_ID';

    $resp = wp_remote_post(
        "https://api.cloudflare.com/client/v4/zones/{$zone_id}/purge_cache",
        [
            'headers' => [
                'Authorization' => 'Bearer ' . $api_key,
                'Content-Type'  => 'application/json',
            ],
            'body'    => wp_json_encode( [ 'purge_everything' => true ] ),
            'timeout' => 20,
        ]
    );

    if ( is_wp_error( $resp ) ) {
        wp_send_json_error( 'Request failed' );
    }

    $code = wp_remote_retrieve_response_code( $resp );
    $body = json_decode( wp_remote_retrieve_body( $resp ), true );

    if ( $code === 200 && ! empty( $body['success'] ) ) {
        wp_send_json_success();
    } else {
        wp_send_json_error( $body ?: 'Unknown error' );
    }
} );
```
