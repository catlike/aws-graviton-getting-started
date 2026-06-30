# Headless website testing with Chrome and Puppeteer on Graviton.

Chrome is a popular reference browser for web site unit testing on EC2 x86 instances. In the samer manner it can be used on EC2 Graviton instances.
It is open source in its Chromium incarnation, supports 'headless' mode and can simulate user actions via Javascript.
The Chrome web site [mentions Puppeteer](https://developer.chrome.com/blog/headless-chrome/#using-programmatically-node) and it is used here to show a specific
example of headless website testing on Graviton.
“[Puppeteer](https://pptr.dev/) is a Node.js library which provides a high-level API to control Chrome/Chromium over the DevTools Protocol. 
Puppeteer runs in headless mode by default, but can be configured to run in full ("headful") Chrome/Chromium."
It can serve as a replacement for the previously very popular Phantomjs.
“PhantomJS (phantomjs.org) is a headless WebKit scriptable with JavaScript. The latest stable release is version 2.1.
Important: PhantomJS development is suspended until further notice (see #15344 for more details).“
The APIs are different, so code targetting PhantomJS has to be rewritten when moving to Puppeteer (see Appendix).
Puppeteer is open source and has 466 contributors and 364k users and thus is likely to be supported for some time.

### Get a recent version of NodeJS.

Ubuntu-22:
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt-get update
sudo apt-get install nodejs -y
```
AL2023:
```
sudo dnf install -y nodejs npm
```

### Install Puppeteer.
```
npm i puppeteer@21.1.0
```
Puppeteer packages an x86 version of Chrome which needs to be replaced with an aarch64 version.

### Install aarch64 version of Chrome.

Ubuntu-22:
```
sudo apt update
sudo apt install chromium-browser chromium-codecs-ffmpeg
```
AL2023 (aarch64):

Google does not publish an aarch64 Linux build of Chrome / Chrome for Testing, and Puppeteer's bundled `linux_arm` download is an x86-64 binary that fails with `Exec format error` on Graviton. Install the headless runtime dependencies, use the aarch64 Chromium build maintained by Playwright, and point Puppeteer at it:

```
# Headless Chromium runtime dependencies
sudo dnf install -y nss atk at-spi2-atk cups-libs libXcomposite \
    libXdamage libXrandr libXext libX11 libxcb mesa-libgbm \
    libxkbcommon pango alsa-lib cairo nspr

# Install Puppeteer without its (x86) bundled Chrome, plus Playwright's browser installer
PUPPETEER_SKIP_DOWNLOAD=1 npm i puppeteer playwright-core

# Download an aarch64 Chromium build (maintained by Playwright)
npx playwright install chromium

# Point Puppeteer at the Playwright-provided Chromium binary
export PUPPETEER_EXECUTABLE_PATH=$(find ~/.cache/ms-playwright -path '*chrome-linux/chrome' | head -1)
```

Launch Puppeteer with this Chromium, for example:

```
const browser = await puppeteer.launch({
  executablePath: process.env.PUPPETEER_EXECUTABLE_PATH,
  headless: 'new',
  args: ['--no-sandbox', '--disable-gpu', '--disable-dev-shm-usage']
});
```

Unlike pinning individual Fedora/CentOS RPMs (which are pruned from their mirrors over time — the previous instructions broke once `qt5-qtbase-5.15.9-2.el9` and the Chromium 116 koji RPMs became unavailable), `npx playwright install chromium` always fetches a current, working aarch64 Chromium for the installed Playwright version.

### Code Examples

Example code for Puppeteer can be found at https://github.com/puppeteer/examples

### Virtual Framebuffer Xserver

Some code examples, such as puppeteer/examples/oopif.js, may need a 'headful' chrome and thus an Xserver.
A virtual framebuffer Xserver can be used for that.

Ubuntu-22.04: ```sudo apt install xvfb```
AL2023: ```sudo yum install Xvfb```
```
Xvfb &
export DISPLAY=:0
```
When Chrome is now invoked in headful mode, it has an Xserver to render to.

This can be tested with:
```
node puppeteer/examples/oopif.js
```
The oopif.js example invokes chrome in headful mode.


## Appendix.

### Code example to show the difference in API between PhantomJS and Puppeteer.


Puppeteer Screenshot:
```
'use strict';

const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://www.google.com', {waitUntil: 'load', timeout: 1000});
  await page.screenshot({path: 'google.png'});
  await browser.close();
})();
```
The same with PhantomJS:
```
var page = require('webpage').create();
page.open('http://www.google.com', function() {
    setTimeout(function() {
        page.render('google.png');
        phantom.exit();
    }, 200);
});
```

