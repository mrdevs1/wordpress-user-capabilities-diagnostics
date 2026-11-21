# WordPress User Roles & Capabilities Diagnostics

A practical troubleshooting guide for diagnosing WordPress user permissions, role assignments, capability checks, and access control problems.

This repository covers default user roles, custom roles, capabilities, permission checks, object-level authorization, role registration, and common access-related issues.

## Table of Contents

- [Overview](#overview)
- [Common Permission Problems](#common-permission-problems)
- [Requirements](#requirements)
- [1. Understand WordPress Roles and Capabilities](#1-understand-wordpress-roles-and-capabilities)
- [2. Inspect User Roles](#2-inspect-user-roles)
- [3. Inspect User Capabilities](#3-inspect-user-capabilities)
- [4. Diagnose Permission Checks](#4-diagnose-permission-checks)
- [5. Check Object-Level Capabilities](#5-check-object-level-capabilities)
- [6. Inspect Registered Roles](#6-inspect-registered-roles)
- [7. Diagnose Custom Role Registration](#7-diagnose-custom-role-registration)
- [8. Check Capability Filters](#8-check-capability-filters)
- [9. Troubleshoot Custom Post Type Permissions](#9-troubleshoot-custom-post-type-permissions)
- [10. Diagnose Multisite Permissions](#10-diagnose-multisite-permissions)
- [11. Check User Role Changes](#11-check-user-role-changes)
- [12. Diagnose Admin Access Problems](#12-diagnose-admin-access-problems)
- [Useful WP-CLI Commands](#useful-wp-cli-commands)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Best Practices](#best-practices)
- [Contributing](#contributing)
- [License](#license)

## Overview

WordPress uses a role and capability system to control what users can do on a website.

A role is a collection of capabilities. Capabilities determine whether a user can perform a particular action.

Common problems include:

- Users cannot access admin pages.
- Editors cannot edit certain content.
- Custom roles have missing permissions.
- Users can access actions they should not perform.
- Capability checks return unexpected results.
- Custom post type permissions do not work.
- Role changes are not reflected correctly.
- Multisite permissions behave differently from single-site permissions.

This repository provides practical diagnostic examples for identifying these issues.

## Common Permission Problems

| Problem | Possible Cause |
|---|---|
| Admin page access denied | Missing capability |
| Editor cannot edit another user's post | Missing edit capability |
| Custom role cannot access a feature | Incorrect role configuration |
| Permission check returns false | Incorrect capability or user context |
| User can perform restricted actions | Excessive capabilities |
| Custom post type permissions fail | Incorrect capability mapping |
| Role changes do not take effect | Registration or assignment issue |
| Multisite access is restricted | Network-level permissions |

## Requirements

- WordPress installation
- Basic PHP knowledge
- Access to an active theme or custom plugin
- WP-CLI (recommended)
- Administrator access for permission testing

Perform permission changes on a staging website before applying them to production.

## 1. Understand WordPress Roles and Capabilities

WordPress provides six predefined roles:

| Role | General Permission |
|---|---|
| Super Admin | Network administration in Multisite |
| Administrator | Site administration |
| Editor | Manage and publish content |
| Author | Publish and manage own posts |
| Contributor | Write and manage own unpublished posts |
| Subscriber | Manage own profile |

A capability represents a specific permission.

Examples:

```text
read
edit_posts
publish_posts
edit_pages
manage_options
upload_files
delete_posts
edit_users
```

Do not assume that one role is simply a higher version of another. Roles are collections of permissions designed for different responsibilities.

Official documentation:

https://wordpress.org/documentation/article/roles-and-capabilities/

## 2. Inspect User Roles

Retrieve a user by ID:

```php
$user = get_userdata( 123 );

if ( $user ) {
    echo 'Username: ' . esc_html( $user->user_login );

    echo PHP_EOL;

    echo 'Roles: ' . esc_html(
        implode( ', ', $user->roles )
    );
}
```

Replace `123` with an actual user ID.

To inspect the current user's roles:

```php
$user = wp_get_current_user();

print_r( $user->roles );
```

Check:

- Whether the user exists.
- Whether the expected role is assigned.
- Whether multiple roles are assigned.
- Whether a custom role is registered correctly.

## 3. Inspect User Capabilities

Use the `WP_User` object to inspect a user's capabilities.

```php
$user = get_userdata( 123 );

if ( $user ) {
    echo '<pre>';

    print_r( $user->allcaps );

    echo '</pre>';
}
```

Check a specific capability:

```php
$user = get_userdata( 123 );

if ( $user && user_can( $user, 'edit_posts' ) ) {
    echo 'User can edit posts.';
} else {
    echo 'User cannot edit posts.';
}
```

The `allcaps` property includes the user's effective capabilities. It can contain capabilities inherited from roles and individual user assignments.

Do not expose this information publicly because it can reveal internal permission configurations.

## 4. Diagnose Permission Checks

WordPress provides `current_user_can()` to check whether the currently logged-in user has a capability.

Example:

```php
if ( current_user_can( 'manage_options' ) ) {
    echo 'Access granted.';
} else {
    echo 'Access denied.';
}
```

Check another user:

```php
$user_id = 123;

if ( user_can( $user_id, 'edit_posts' ) ) {
    echo 'Permission granted.';
}
```

For authorization checks, prefer capability names instead of checking role names directly.

Avoid:

```php
if ( current_user_can( 'administrator' ) ) {
    // Avoid checking a role as if it were a capability.
}
```

Prefer:

```php
if ( current_user_can( 'manage_options' ) ) {
    // Check the actual required capability.
}
```

This makes permission checks more flexible when custom roles are introduced.

## 5. Check Object-Level Capabilities

Some WordPress capabilities depend on a specific object.

For example, permission to edit one post can differ from permission to edit all posts.

Check a post:

```php
$post_id = 123;

if ( current_user_can( 'edit_post', $post_id ) ) {
    echo 'You can edit this post.';
} else {
    echo 'You cannot edit this post.';
}
```

Other examples:

```php
current_user_can( 'delete_post', $post_id );

current_user_can( 'publish_posts' );

current_user_can( 'edit_others_posts' );
```

WordPress maps meta capabilities such as `edit_post` to the primitive capabilities required for the particular object.

This distinction is important when troubleshooting ownership and editing restrictions.

## 6. Inspect Registered Roles

Use `wp_roles()` to inspect registered roles.

```php
$roles = wp_roles()->roles;

foreach ( $roles as $role_key => $role_data ) {

    echo 'Role: ' . $role_key . PHP_EOL;

    echo 'Name: ' . $role_data['name'] . PHP_EOL;

    echo 'Capabilities:' . PHP_EOL;

    print_r( $role_data['capabilities'] );

    echo PHP_EOL;
}
```

You can also inspect a specific role:

```php
$role = get_role( 'editor' );

if ( $role ) {
    print_r( $role->capabilities );
}
```

Check whether:

- The expected role exists.
- Required capabilities are present.
- Unexpected capabilities have been added.
- A plugin has modified role permissions.

## 7. Diagnose Custom Role Registration

Custom roles can be registered using `add_role()`.

Example:

```php
add_action( 'init', function () {

    add_role(
        'content_reviewer',
        'Content Reviewer',
        array(
            'read'         => true,
            'edit_posts'   => true,
            'edit_pages'   => true,
            'upload_files' => true,
        )
    );

} );
```

Important considerations:

- `add_role()` does not overwrite an existing role with the same slug.
- Register roles during plugin activation or another controlled setup process.
- Do not repeatedly add capabilities on every request.
- Remove custom roles during plugin deactivation or uninstall only when appropriate.

For persistent plugin-based role registration, an activation hook is generally preferable:

```php
register_activation_hook( __FILE__, function () {

    add_role(
        'content_reviewer',
        'Content Reviewer',
        array(
            'read'       => true,
            'edit_posts' => true,
        )
    );

} );
```

When changing an existing role's capabilities, use `add_cap()` or `remove_cap()` as appropriate.

## 8. Check Capability Filters

WordPress provides filters that can modify permission behavior.

Important hooks include:

| Hook | Purpose |
|---|---|
| `user_has_cap` | Modify a user's effective capabilities |
| `map_meta_cap` | Modify meta capability mapping |
| `role_has_cap` | Filter capabilities assigned through a role |

Example of inspecting effective capabilities:

```php
add_filter(
    'user_has_cap',
    function ( $allcaps, $caps, $args, $user ) {

        if ( defined( 'WP_DEBUG' ) && WP_DEBUG ) {
            error_log(
                'Capability check for user: ' . $user->ID
            );
        }

        return $allcaps;
    },
    10,
    4
);
```

When a permission check returns an unexpected result:

1. Review custom permission filters.
2. Check active membership and role management plugins.
3. Inspect capability assignments.
4. Test with a default theme on staging.
5. Compare the results with plugins temporarily disabled.

Never modify capability arrays without understanding the impact on other permission checks.

## 9. Troubleshoot Custom Post Type Permissions

Custom post types can define their own capability settings.

Example:

```php
register_post_type(
    'project',
    array(
        'label'        => 'Projects',
        'public'       => true,
        'show_ui'      => true,
        'capability_type' => array(
            'project',
            'projects',
        ),
        'map_meta_cap' => true,
        'supports'     => array(
            'title',
            'editor',
        ),
    )
);
```

Inspect the registered post type:

```php
$post_type = get_post_type_object( 'project' );

if ( $post_type ) {
    print_r( $post_type->cap );
}
```

Check:

- The post type's capability configuration.
- Whether `map_meta_cap` is enabled when required.
- Whether the role has the necessary capabilities.
- Whether custom capabilities are registered and assigned.
- Whether the user owns the content being edited.

A custom post type's capability configuration must match the permissions assigned to the intended roles.

## 10. Diagnose Multisite Permissions

WordPress Multisite has additional network-level permission rules.

A Super Admin can manage network-level features, while individual site administrators have permissions limited by the network configuration.

Check whether the installation is Multisite:

```php
if ( is_multisite() ) {
    echo 'Multisite is enabled.';
} else {
    echo 'Single-site installation.';
}
```

Check Super Admin status:

```php
$user_id = 123;

if ( is_super_admin( $user_id ) ) {
    echo 'User is a Super Admin.';
} else {
    echo 'User is not a Super Admin.';
}
```

Investigate:

- Network administration permissions.
- Site-specific user roles.
- Plugin and theme installation restrictions.
- User membership on individual sites.
- Network-level capability restrictions.

Do not assume that a site Administrator has all the permissions of a Multisite Super Admin.

## 11. Check User Role Changes

WordPress provides methods for modifying user roles.

Set a user's primary role:

```php
$user = new WP_User( 123 );

$user->set_role( 'editor' );
```

Add another role:

```php
$user->add_role( 'author' );
```

Remove a role:

```php
$user->remove_role( 'author' );
```

These operations modify the user's role assignments.

Use them carefully. `set_role()` replaces the user's existing role assignments, while `add_role()` and `remove_role()` modify individual role assignments.

Verify the result:

```php
$user = get_userdata( 123 );

if ( $user ) {
    print_r( $user->roles );
}
```

Always verify the user ID and intended role before applying changes.

## 12. Diagnose Admin Access Problems

A user may log in successfully but still be unable to access certain administration screens.

Check:

1. Whether the user is authenticated.
2. Whether the expected role is assigned.
3. Whether the required capability exists.
4. Whether a plugin restricts admin access.
5. Whether custom code changes capability checks.
6. Whether the website uses Multisite.
7. Whether the target admin page performs an additional permission check.

Example:

```php
add_action( 'admin_init', function () {

    if ( ! current_user_can( 'manage_options' ) ) {
        return;
    }

    // Administrator-level functionality.

} );
```

For a custom admin page, use an appropriate capability when registering the menu:

```php
add_menu_page(
    'Project Settings',
    'Project Settings',
    'manage_options',
    'project-settings',
    'render_project_settings'
);
```

The capability should match the intended audience of the page.

## Useful WP-CLI Commands

WP-CLI provides commands for inspecting WordPress users.

| Command | Purpose |
|---|---|
| `wp user list` | List WordPress users |
| `wp user get 123` | Inspect a user |
| `wp user get 123 --field=roles` | Display assigned roles |
| `wp role list` | List registered roles |
| `wp role get editor` | Inspect a role |
| `wp cap list editor` | List role capabilities |
| `wp help user` | Display user command help |
| `wp help role` | Display role command help |

Replace example user IDs and role names with actual values.

Before modifying roles or capabilities, take a database backup.

## Troubleshooting Checklist

Use this checklist when diagnosing WordPress permission problems.

- [ ] Confirm the user account exists.
- [ ] Verify the assigned user roles.
- [ ] Inspect effective user capabilities.
- [ ] Check the required capability for the action.
- [ ] Use object-level checks when necessary.
- [ ] Inspect registered custom roles.
- [ ] Verify custom role registration.
- [ ] Review capability filters.
- [ ] Check custom post type capability settings.
- [ ] Verify Multisite restrictions.
- [ ] Review role management plugins.
- [ ] Test permission behavior on staging.
- [ ] Back up the database before making changes.

## Best Practices

- Use capability checks instead of role-name checks.
- Follow the principle of least privilege.
- Assign only the permissions users actually need.
- Avoid granting `manage_options` unnecessarily.
- Use object-level capabilities for protected content.
- Keep custom role registration predictable.
- Avoid changing core WordPress role definitions directly.
- Review plugin permission filters when debugging access issues.
- Test permission changes with accounts representing each role.
- Back up the database before bulk permission changes.
- Never expose internal capability information publicly.

## Contributing

Contributions are welcome.

You can help improve this repository by:

- Reporting permission-related bugs.
- Adding diagnostic examples.
- Improving role and capability documentation.
- Documenting custom post type permission issues.
- Sharing reproducible access control problems.

When reporting an issue, include:

- WordPress version
- PHP version
- User role
- Required capability
- Expected behavior
- Actual behavior
- Relevant plugin or theme information

Do not include passwords, private user information, or authentication tokens.

## License

This project is licensed under the MIT License.

You are free to use, modify, and distribute this material with appropriate attribution.

---

**WordPress User Roles & Capabilities Diagnostics**

A practical reference for troubleshooting WordPress user permissions and access control.
