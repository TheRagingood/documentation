# Themes

Themes allow you to customize the look and feel of the Standard Notes app on all platforms.

## Web/Desktop Themes

You can view the [source code](https://github.com/sn-extensions/themes/tree/master/themes) of our official themes in order to best understand how to create your own theme.

Themes are simply css files that are installed with the following link format:

```
https://host.org/theme.css?type=theme&name=MyTheme
```

Note that if you are using the official Standard Notes web app, your theme must be hosted behind an HTTPS link or else your browser will refuse to load it.

## Mobile Themes

Mobile themes offer control mostly on colors. They are simply JSON files hosted on a server.

### JSON Format:

```
{
  name: "Midnight",
  constants: {
    mainTintColor: "#fffdff",
    mainBackgroundColor: "#171925",
    mainTextColor: "#bbbdcb",
    navBarColor: "#171925",
    mainDimColor: "#363b53",
    navBarTextColor: "#fffdff",
    plainCellBorderColor: "#1d1f2f",
    selectedBackgroundColor: "#292b3a",
    composeBorderColor: "#1b1b1b"
  },

  rules: {

  },

  statusBar: "light-content"
}
```

Mobile themes are installed the same way regular themes are installed. In fact, when you install a regular css theme, the mobile app will use the same link, but replace `.css` with `.json`:

When you install

```
https://host.org/theme.css?type=theme&name=MyTheme
```

the mobile app will look for

```
https://host.org/theme.json?type=theme&name=MyTheme
```

and automatically display it in the themes list. If the server does not respond to the url, the mobile app will disable that theme and gray it out in the list.

### Reloading Mobile Themes

The mobile app will download a theme once and cache it indefinitely. If you're installing your own mobile theme and make changes, you can press and hold on the theme name in the list to bring up the option to re-download the theme from the server.

# Licensing

Our themes are provided open-source mainly for educational and quality purposes. You're free to install them on your own servers, but please consider subscribing to [Standard Notes Extended](https://standardnotes.org/extended) to help sustain future development and the entire Standard Notes ecosystem.
