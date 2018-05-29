
# Create a Native Application for any Website

When I first found [this package](https://github.com/jiahaog/nativefier) on Github, I thought I had found a new way to create Electron apps (completing missing the fact that the page has nearly 15000 stars). *Nativefier* is a tool that allows you to create a native application on Windows, MacOS, or Linux using the command line with minimal configuration.

Not too long ago, developers disliked Electron apps because they were slower, less power efficient, used a ton of memory, and didn't offer the same features as native applications. Now, many companies use Electron who still manage to provide an incredible user experience, such as Slack and WhatsApp.

### Installation
First, add Nativefier globally (*-g* flag) using NPM (Node Package Manager)
```bash
npm install nativefier -g
```

### How to Use
Here's a basic introduction:
```bash
nativefier 'https://app.timber.io/auth/login'
```

***Yeah, it's seriously that easy***
You can use a few flags such as `--name "Timber` or `--icon /src/to/icon.png` to customize your installation, but Nativefier automatically tries to scrape that information from the website.

There are still downsides with using Nativefier to create your Electron Applications. It doesn't provide the same user experience as creating the application from scratch, so I wouldn't recommend it for developers who are looking to create a professional application. If you're just looking to create a web app of one of your favorite websites because you hate having to use `⌘-tab` to switch back to the tab, it's really a great tool.
