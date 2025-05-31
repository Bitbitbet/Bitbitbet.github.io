I'll put something that I recognize as hard-to-memorize here during gtk-rs development.

## Crates to add

`gtk4`, the main crate for gtk4 development. Check features of gtk4 [here](https://crates.io/crates/gtk4#user-content-features-1).
```sh
cargo add gtk4 --rename gtk --features v4_18
```
---
`libadwaita`, the crate that adds modern look and feel to gtk. Check features of libadwaita [here](https://crates.io/crates/libadwaita/versions)
```sh
cargo add libadwaita --rename adw --features v1_7
```
---
`glib-build-tools`, the crate that should be added as a build dependency to allow integration for gresources.
```sh
cargo add glib-build-tools --build
```

## Gresources
### Adding the build script `build.rs`
The build script is responsible for integration of gresources integration during compilation. `glib-build-tools` should be added as a build dependency for this to work.

After adding `glib-build-tools`, create `build.rs` at the root of the project and add the following content to it:
```rust
fn main() {
    glib_build_tools::compile_resources(
        &["resources"],
        "resources/resources.gresources.xml",
        "compiled.gresource",
    );
}
```
And then register the gresources in the real code, typically main function in `src/main.rs`:
```rust
fn main() -> glib::ExitCode {
    gio::resources_register_include!("compiled.gresource").expect("Failed to register gresources");

    let app = Application::builder().application_id(/* Your application ID */).build();
    app.connect_activate(activate);
    app.run()
}
```
Now gtk will access `/resources/resources.gresources.xml` and compile all resources specified by it during build time. Example of `resources.gresources.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<gresources>
	<gresource prefix="/org/application/example">
		<file compressed="true" preprocess="xml-stripblanks">main_window.ui</file>
		<file compressed="true" preprocess="xml-stripblanks">icons/right-small-symbolic.svg</file>
	</gresource>
</gresources>
```
## Creating custom GObject
Every custom gobject requires two structs to construct. One of them is defined through marcro `glib::wrapper!`, and the other one is called *implementation struct* and is defined through Rust's struct syntax, and typically with `Imp` suffixing the name.

The simplist custom gobject could be defined like this:
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>);
    //                                                      Semicolon here -- ^
}

mod custom_gobject_imp {
    #[derive(Default)]
    struct CustomGObjectImp {}

    #[glib::object_subclass]
    impl ObjectSubclass for CustomGObjectImp {
        const NAME: &'static str = "ExampleApplicationCustomGObject";
        type Type = super::CustomGObject;
    }

    #[gtk::template_callbacks]
    impl CustomGObjectImp {}

    #[glib::derived_properties]
    impl ObjectImpl for CustomGObjectImp {}
}
```
This is a gobject named `CustomGObject`, and is identified as `ExampleApplicationCustomGObject` by glib at runtime.
### Add properties
Custom gobjects with properties could be defined as follows:
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>);
}

mod custom_gobject_imp {
    #[derive(Default, Properties)]
    #[properties(wrapper_type = super::CustomGObject)]
    struct CustomGObjectImp {
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
Custom derive macro `Properties` is added to the imp struct, followed by another macro `properties` to specifiy where methods like `custom_property`, `set_custom_property`, `connect_custom_property_notify` go to. This should be set to the struct defined through `glib::wrapper!` macro.

Note that properties should be defined with interior mutability if you want it to be mutable, and use `self.obj().set_custom_property(...)` to change its value instead of direct manipulating so that the change could be noticed by glib.

### Subclassing
Custom gobjects can extend one of the existing gobjects inside gtk, libadwaita or glib etc., e.g.:

```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>)
        @extends /* list of gobjects to be extended, seperated by comma with trailing comma, but without semicolon at the end */
        @implements /* list of interfaces to be implemented, seperated by comma without trailing comma */;
    // Semicolon is moved to the @implements line
}

mod custom_gobject_imp {
    #[derive(Default)]
    struct CustomGObjectImp {}

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

The list of objects and interfaces inside `@extends` and `@implements` could be referenced at documentation of the direct parent. For example, list of objects and interfaces that `adw::ApplicationWindow` has is documented at [here](https://gnome.pages.gitlab.gnome.org/libadwaita/doc/1.0/class.ApplicationWindow.html#hierarchy). So to make our CustomGObject extend `adw::ApplicationWindow`, it should be defined like this:
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>)
        @extends adw::ApplicationWindow, gtk::ApplicationWindow, gtk::Window, gtk::Widget
        @implements gio::ActionGroup, gio::ActionMap, gtk::Accessible, gtk::Buildable, gtk::ConstraintTarget, gtk::Native, gtk::Root, gtk::ShortcutManager;
}
```
Note that no need to add `glib::Object` and `glib::InitiallyUnowned` in `@extends`

### Use Composite Template
Composite template is very useful in defining UI.
```rust
glib::wrapper! {
    pub struct CustomGObject(ObjectSubclass<custom_gobject_imp::CustomGObjectImp>)
        @extends adw::ApplicationWindow, gtk::ApplicationWindow, gtk::Window, gtk::Widget
        @implements gio::ActionGroup, gio::ActionMap, gtk::Accessible, gtk::Buildable, gtk::ConstraintTarget, gtk::Native, gtk::Root, gtk::ShortcutManager;
}

mod custom_gobject_imp {
    #[derive(Default, CompositeTemplate)]
    #[template(resource = "/org/application/example/main_window.ui")]
    struct CustomGObjectImp {}

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

    #[glib::derived_properties]
    impl ObjectImpl for CustomGObjectImp {}
    impl WidgetImpl for MainWindowImp {}
    impl WindowImpl for MainWindowImp {}
    impl ApplicationWindowImpl for MainWindowImp {}
    impl AdwApplicationWindowImpl for MainWindowImp {}
}
```

---
Below is a custom gobject named CustomGObject, with CompositeTemplate and custom properties.
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
    struct CustomGObjectImp {}

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