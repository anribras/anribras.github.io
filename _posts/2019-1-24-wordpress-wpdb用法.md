---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

<!-- TOC -->

- [delete meta表里某个](#delete-meta表里某个)
- [delete整个表](#delete整个表)
- [](#)

<!-- /TOC -->

## delete meta表里某个

meta_key: _liked 

```php
    $do_action 		= $wpdb->delete($meta_table, array( 'meta_key' => $meta_key ));

    if ($do_action === FALSE) {
        wp_send_json_error( __( 'Failed! An Error Has Occurred While Deleting All ULike Logs/Data', WP_ULIKE_SLUG ));
    } else {
        wp_send_json_success( __( 'Success! All ULike Logs/Data Have Been Deleted', WP_ULIKE_SLUG ) );
    }
```

##  delete整个表
```php
    if ($wpdb->query("TRUNCATE TABLE $logs_table") === FALSE) {
			wp_send_json_error( __( 'Failed! An Error Has Occurred While Deleting All ULike Logs/Data', WP_ULIKE_SLUG ) );
		} else {
			wp_send_json_success( __( 'Success! All ULike Logs/Data Have Been Deleted', WP_ULIKE_SLUG ) );
		}
```

## 