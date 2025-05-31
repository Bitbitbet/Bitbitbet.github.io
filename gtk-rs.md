I'll put something that I recognize as hard-to-memorize here during gtk-rs development.

## Common Crates Used

`gtk4`, the main crate for gtk4 development. Check features of gtk4 [here](https://crates.io/crates/gtk4#user-content-features-1).
```sh
cargo add gtk4 --rename gtk --features v4_18
```
---
`libadwaita`, a crate that adds modern look and feel to gtk. Check features of libadwaita [here](https://crates.io/crates/libadwaita/versions)
```sh
cargo add libadwaita --rename adw --features v1_7
```
---
`glib-build-tools`, the crate that allows integration for gresources and should be added as a build dependency.
```sh
cargo add glib-build-tools --build
```

## Gresources
### Adding the build script `build.rs`
The build script is responsible for integration of gresources integration during compilation. `glib-build-tools` should be added as a build dependency for this to work.

After adding `glib-build-tools`, create `build.rs` at the root of the project with the following content:
```rust
// /build.rs

fn main() {
    glib_build_tools::compile_resources(
        &["resources"],
        "resources/resources.gresources.xml",
        "compiled.gresource",
    );
}
```
And then register the gresources, normally at `main` at `src/main.rs`:
```rust
// /src/main.rs

fn main() -> glib::ExitCode {
    gio::resources_register_include!("compiled.gresource").expect("Failed to register gresources");

    let app = Application::builder().application_id(/* Your application ID */).build();
    app.connect_activate(activate);
    app.run()
}
```
Now gtk will access `/resources/resources.gresources.xml` and compile all resources specified by it during build time. Example of `resources.gresources.xml`:
```xml
<!-- /resources/resources.gresources.xml -->

<?xml version="1.0" encoding="UTF-8"?>
<gresources>
	<gresource prefix="/org/application/example">
		<file compressed="true" preprocess="xml-stripblanks">main_window.ui</file>
		<file compressed="true" preprocess="xml-stripblanks">icons/right-small-symbolic.svg</file>
	</gresource>
</gresources>
```
## Creating Custom GObject
Every custom gobject requires two structs to construct. One of them is defined through marcro `glib::wrapper!`, and the other one is called *implementation struct* and is defined through Rust's struct syntax, and typically with `Imp` suffixing the name.

Here is the simplist custom gobject:
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>);
    //                                                          Semicolon here -- ^
}

mod custom_gobject_imp {
    #[derive(Default)]
    pub struct CustomGObjectImp {}

    #[glib::object_subclass]
    impl ObjectSubclass for CustomGObjectImp {
        const NAME: &'static str = "ExampleApplicationCustomGObject";
        type Type = super::CustomGObject;
    }

    impl CustomGObjectImp {}

    impl ObjectImpl for CustomGObjectImp {}
}
```
Above defines a gobject named `CustomGObject`, and is identified as `ExampleApplicationCustomGObject` by glib at runtime.
### Add properties
Here is a custom gobject with custom properties:
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>);
}

mod custom_gobject_imp {
    #[derive(Default, Properties)]
    #[properties(wrapper_type = super::CustomGObject)]
    pub struct CustomGObjectImp {
        #[property(get, set)]
        custom_property: RefCell<String>,
    }

    #[glib::object_subclass]
    impl ObjectSubclass for CustomGObjectImp {
        const NAME: &'static str = "ExampleApplicationCustomGObject";
        type Type = super::CustomGObject;
    }

    impl CustomGObjectImp {}

    #[glib::derived_properties]
    impl ObjectImpl for CustomGObjectImp {}
}
```
Custom derive macro `Properties` is added to the imp struct, followed by another macro `properties` to specifiy where methods like `custom_property`, `set_custom_property`, `connect_custom_property_notify` go to. `wrapper_type` should be set to the struct defined through `glib::wrapper!` macro.

Note that properties should be defined with interior mutability if you want it to be mutable, and use `self.obj().set_custom_property(...)` to change its value instead of direct manipulating so that the change could be noticed by glib.

### Subclassing
Custom gobjects can extend one of the existing gobjects in gtk, libadwaita or glib etc.:

```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>)
        @extends /* list of gobjects to be extended, seperated by comma with trailing comma, but without semicolon at the end */
        @implements /* list of interfaces to be implemented, seperated by comma without trailing comma */;
    // Semicolon is moved to the @implements line
}

mod custom_gobject_imp {
    #[derive(Default)]
    pub struct CustomGObjectImp {}

    #[glib::object_subclass]
    impl ObjectSubclass for CustomGObjectImp {
        const NAME: &'static str = "ExampleApplicationCustomGObject";
        type Type = super::CustomGObject;
        type ParentType = /* the direct parent. could be omitted if it is glib::Object */;
    }

    impl CustomGObjectImp {}

    impl ObjectImpl for CustomGObjectImp {}
}
```

The list of objects and interfaces that should be specified inside `@extends` and `@implements` are documented at the official documentation of the direct parent. For example, list of objects and interfaces that `adw::ApplicationWindow` has is documented [here](https://gnome.pages.gitlab.gnome.org/libadwaita/doc/1.0/class.ApplicationWindow.html#hierarchy). So here's how to make our CustomGObject extend `adw::ApplicationWindow`:
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>)
        @extends adw::ApplicationWindow, gtk::ApplicationWindow, gtk::Window, gtk::Widget
        @implements gio::ActionGroup, gio::ActionMap, gtk::Accessible, gtk::Buildable, gtk::ConstraintTarget, gtk::Native, gtk::Root, gtk::ShortcutManager;
}
```
And implement several `Impl`s for the imp struct:
```rust
mod custom_gobject_imp {

    /* ... */

    impl ObjectImpl for CustomGObjectImp {}
    impl WidgetImpl for MainWindowImp {}
    impl WindowImpl for MainWindowImp {}
    impl ApplicationWindowImpl for MainWindowImp {}
    impl AdwApplicationWindowImpl for MainWindowImp {}
    
    /* ... */

}

```

Note that there's no need to add `glib::Object` and `glib::InitiallyUnowned` in `@extends`

### Use Composite Template
Composite template is very useful in constructing UI. Here's how to associate `CustomGObject` with a template file named `main_window.rs`:
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>)
        @extends adw::ApplicationWindow, gtk::ApplicationWindow, gtk::Window, gtk::Widget,
        @implements gio::ActionGroup, gio::ActionMap, gtk::Accessible, gtk::Buildable, gtk::ConstraintTarget, gtk::Native, gtk::Root, gtk::ShortcutManager;
}

mod custom_gobject_imp {
    #[derive(Default, CompositeTemplate)]
    #[template(resource = "/org/application/example/main_window.ui")]
    pub struct CustomGObjectImp {}

    #[glib::object_subclass]
    impl ObjectSubclass for CustomGObjectImp {
        const NAME: &'static str = "ExampleApplicationCustomGObject";
        type Type = super::CustomGObject;
        type ParentType = adw::ApplicationWindow;

        fn class_init(klass: &mut Self::Class) {
            klass.bind_template();
            klass.bind_template_callbacks();
        }

        fn instance_init(obj: &InitializingObject<Self>) {
            obj.init_template();
        }
    }

    #[gtk::template_callbacks]
    impl CustomGObjectImp {}

    impl ObjectImpl for CustomGObjectImp {}
    impl WidgetImpl for MainWindowImp {}
    impl WindowImpl for MainWindowImp {}
    impl ApplicationWindowImpl for MainWindowImp {}
    impl AdwApplicationWindowImpl for MainWindowImp {}
}
```
Now CustomGObject will follow the template file during the construction process.
#### UI template file
`main_window.ui` should be defined inside `resources.gresources.xml`:
```xml
<file compressed="true" preprocess="xml-stripblanks">main_window.ui</file>
```
and here's what `main_window.ui` may look like:
```xml
<!-- /resources/main_window.ui -->

<?xml version="1.0" encoding="UTF-8"?>
<interface>
	<template class="ExampleApplicationCustomGObject" parent="AdwApplicationWindow">
		<property name="content">
			<object class="GtkButton">
				<property name="label">A Button</property>
			</object>
		</property>
	</template>
</interface>
```

---
Below is CustomGObject with CompositeTemplate and custom properties.
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>)
        @extends /* list of gobjects to be extended, seperated by comma with trailing comma, but without semicolon at the end */
        @implements /* list of interfaces to be implemented, seperated by comma without trailing comma */;
}

mod custom_gobject_imp {
    #[derive(Default, CompositeTemplate, Properties)]
    #[properties(wrapper_type = super::CustomGObject)]
    #[template(resource = "/org/application/example/main_window.ui")]
    pub struct CustomGObjectImp {}

    #[glib::object_subclass]
    impl ObjectSubclass for CustomGObjectImp {
        const NAME: &'static str = "ExampleApplicationCustomGObject";
        type Type = super::CustomGObject;
        type ParentType = /* the direct parent. could be omitted if it is glib::Object */;

        fn class_init(klass: &mut Self::Class) {
            klass.bind_template();
            klass.bind_template_callbacks();
        }

        fn instance_init(obj: &InitializingObject<Self>) {
            obj.init_template();
        }
    }

    #[gtk::template_callbacks]
    impl CustomGObjectImp {}

    #[glib::derived_properties]
    impl ObjectImpl for CustomGObjectImp {}
}
```
