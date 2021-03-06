---
description: 'Adding Badges, Progress Bars, and Quick Lists'
---

# Launchers

Applications can show additional information in the dock as well as the application menu. This makes the application feel more integrated into the system and give user it's status at a glance. See [HIG for Dock integration](https://elementary.io/docs/human-interface-guidelines#dock-integration) for what you should do and what you shouldn't.

For this integration you can use [Unity's Launcher API](https://valadoc.org/unity/Unity.LauncherEntry.html). This API is used accross many different distributions and is widely supported by third party applications. The [Launcher API documentation](https://wiki.ubuntu.com/Unity/LauncherAPI) also provides how the library works internally as well as implementation examples for Python and C if you wish to use any other language for your application than Vala.

#### Current Launcher API support:

| Service | Badge Counter | Progress Bar | Static Quicklist | Dynamic Quicklist |
| :--- | :--- | :--- | :--- | :--- |
| Application menu | Yes | No | Yes | No |
| Dock | Yes | Yes | Yes | Yes |

## Setting Up

Before writing the code, you must first install the `libunity` library, you can do it by executing the following command in Terminal:

```bash
sudo apt install libunity-dev
```

Now let's add the Unity library to your build system. Open your meson.build file and add the new dependency to the `executable` method.

```text
executable(
    meson.project_name(),
    'src/Application.vala',
    dependencies: [
        dependency('gtk+-3.0'),
        dependency('unity')
    ],
    install: true
)
```

Though we haven't made any changes to our source code yet, change into your build directory and run `ninja` to build your project. It should still build without any errors. If you do encounter errors, double check your changes and resolve them before continuing.

## Using the Launcher API

Once you've set up `libunity` in your build system it's time to write some code.

The first thing you'll need to use the API is your application desktop ID. This is usually the filename of your application entry that is installed in the `/usr/share/applications` directory like: `com.github.username.application.desktop`. Keep in mind that if you're generating the desktop file with a build system, the **desktop ID is the final basename of the file generated by your build system** and not a string ending with `.in` or any other type of extension.

You can retrieve a new [Unity.LauncherEntry](https://valadoc.org/unity/Unity.LauncherEntry.html) instance by calling a static `Unity.LauncherEntry.get_for_desktop_id` function:

```csharp
var entry = Unity.LauncherEntry.get_for_desktop_id ("my-desktop-id.desktop");
```

This entry instance allows you to modify your entry so that it shows additional information e.g: on the dock. It is up to you, where in the code you want to retrieve this entry, the function is static so there is no problem accessing it. Usually it's your application or main window class.

Showing a `12` number in the badge is as easy as:

```csharp
entry.count_visible = true;
entry.count = 12;
```

Keep in mind you have to set the `count_visible` property to true, and use an int64 type for the `count` property.

The same goes for showing a progress bar, here we show a progress bar showing 20% progress:

```csharp
entry.progress_visible = true;
entry.progress = 0.2f;
```

As you can see the type of `progress` property is `double` and is a range between `0` and `1`: from 0% to 100%.

## Dynamic Quicklists

Dynamic quicklists are a way to provide the user with dynamic quick menu entries to access some kind of feature in your app. These are shown e.g: right-clicking an open instance of the settings app in the dock. Note that dynamic menu entries can be only provided by a **running** application or processes. **If you always want to expose quick actions in e.g: the Applications Menu**, see [Static Quicklists](launchers.md#static-quicklists).

Here's a simple example of how to make use of dynamic quicklists in Vala:

```csharp
// Create a root quicklist
var quicklist = new Dbusmenu.Menuitem ();

// Create root's children
var item1 = new Dbusmenu.Menuitem ();
item1.property_set (Dbusmenu.MENUITEM_PROP_LABEL, "Item 1");
item1.item_activated.connect (() => {
    message ("Item 1 activated");
});

var item2 = new Dbusmenu.Menuitem ();
item2.property_set (Dbusmenu.MENUITEM_PROP_LABEL, "Item 2");
item2.item_activated.connect (() => {
    message ("Item 2 activated");
});

// Add children to the quicklist
quicklist.child_append (item1);
quicklist.child_append (item2);

// Finally, tell libunity to show the desired quicklist
entry.quicklist = quicklist;
```

Please see the [Dbusmenu.Menuitem API](https://valadoc.org/dbusmenu-glib-0.4/Dbusmenu.Menuitem.html) for more details and features.

## Static Quicklists

The main difference between dynamic and static quicklists is that static ones cannot be changed at runtime. Static quicklists do not involve writing any code or using any external dependencies.

Static quicklists are stored within your `.desktop` file. These are so called "actions". You can define many actions in your desktop file that will always show as an action in the application menu as well as in the dock.

The format is as follows:

```text
[Desktop Action ActionID]
Name=The name of the action
Icon=The icon of the action (optional)
Exec=The path to application executable and it's command line arguments (optional)
```

Let's take a look at an example of an action that will open a new window of your application:

```text
[Desktop Entry]
Name=Application name
Exec=application-executable
...

[Desktop Action NewWindow]
Name=New Window
Exec=application-executable -n
```

Note that adding `-n` or any other argument will not make your application magically open a new window. It is up to your application to handle and interpret command line arguments. The [GLib.Application API](https://valadoc.org/gio-2.0/GLib.Application.html) provides many examples and an extensive documentation on how to handle these arguments, particularly the [command\_line signal](https://valadoc.org/gio-2.0/GLib.Application.command_line.html).

Please take a look at a [freedesktop.org Additional applications actions section](https://standards.freedesktop.org/desktop-entry-spec/latest/ar01s10.html) for a detailed description of what keys are supported and what they do.

