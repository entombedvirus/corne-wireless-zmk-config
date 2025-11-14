USER: I have two split keyboards with the same board and shield (nice_nano_v2 and corne) but one has rgb leds and has different display modules (nice_view and nice_oleds). How can I set up my repo so that the keymap is shared but I can provide different device config to each keybaord>
3 hours ago

AGENT: To configure two similar keyboards with shared keymaps but different device settings in ZMK, you should use separate build configurations for each one. The key is to manage the shared keymap and the device-specific Kconfig and devicetree overlays in different folders. This can all be handled within a single zmk-config repository.

Repo structure
Organize your config folder to handle multiple keyboards and their specific overlays. For this scenario, you could use a structure like this:
zmk-config/
├── config/
│   ├── keyboards/
│   │   ├── corne_rgb/
│   │   │   ├── corne_rgb.conf
│   │   │   └── corne_rgb.overlay
│   │   └── corne_oled/
│   │       ├── corne_oled.conf
│   │       └── corne_oled.overlay
│   ├── common.dtsi
│   ├── build.yaml
│   └── boards/
│       └── shields/
│           ├── corne_left.overlay
│           └── corne_right.overlay
├── .github/
│   └── workflows/
│       └── build.yml

1. Create a common keymap (common.dtsi)
Create a keymap file with your shared layer definitions. This is the heart of your common configuration and should not contain any hardware-specific settings.
Example: config/common.dtsi
dtsi
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/rgb.h>

/ {
    keymap {
        compatible = "zmk,keymap";
        default_layer {
            bindings = <
                &kp Q  &kp W  &kp E  &kp R  &kp T        &kp Y  &kp U  &kp I  &kp O  &kp P
                &kp A  &kp S  &kp D  &kp F  &kp G        &kp H  &kp J  &kp K  &kp L  &kp SQT
                &kp Z  &kp X  &kp C  &kp V  &kp B        &kp N  &kp M  &kp COMMA &kp DOT &kp FSLH
                &trans &trans &mo 1  &kp BSPC &kp TAB    &kp RET &mo 2  &trans &trans &trans
            >;
        };

        // Define your other layers here.
        // ...
    };
};
Use code with caution.

2. Define keyboard-specific overlay files
Create distinct .overlay files for each keyboard, which include the common keymap but override settings for the unique features.
For the RGB + OLED keyboard
Example: config/keyboards/corne_rgb/corne_rgb.overlay
dtsi
#include "../common.dtsi"

// Include standard corne overlays
#include <corne_left.overlay>
#include <corne_right.overlay>

// Enable RGB underglow and a display.
/ {
    chosen {
        zmk,rgb_underglow = &rgb_underglow;
        zmk,display = &nice_oled;
    };
};
Use code with caution.

For the Nice!View + No RGB keyboard
Example: config/keyboards/corne_oled/corne_oled.overlay
dtsi
#include "../common.dtsi"

// Include standard corne overlays
#include <corne_left.overlay>
#include <corne_right.overlay>

// The key here is to remove or override the RGB settings
// and enable the nice!view shield.
/ {
    chosen {
        // We do not define zmk,rgb_underglow here
        zmk,display = &nice_view;
    };
};

// Also define the nice!view shield if needed
#include <nice_view_adapter.overlay>
#include <nice_view.overlay>
Use code with caution.

3. Add keyboard-specific Kconfig files
To enable features like RGB and different displays, you need to set the appropriate CONFIG values in Kconfig files.
Example: config/keyboards/corne_rgb/corne_rgb.conf
kconfig
# Enable RGB Underglow
CONFIG_ZMK_RGB_UNDERGLOW=y
CONFIG_WS2812_STRIP=y

# Enable OLED support
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_ZMK_WIDGET_BATTERY_STATUS=y
CONFIG_ZMK_WIDGET_BATTERY_STATUS_SHOW_PERCENTAGE=y
Use code with caution.

Example: config/keyboards/corne_oled/corne_oled.conf
kconfig
# Do NOT enable RGB.

# Enable display support for Nice!View
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_WIDGET_BATTERY_STATUS=y
CONFIG_ZMK_WIDGET_BATTERY_STATUS_SHOW_PERCENTAGE=y
Use code with caution.

4. Update the build file
Finally, modify your .github/workflows/build.yml file to build firmware for each keyboard and its specific configuration.
Example: .github/workflows/build.yml
yaml
# ... (existing workflow)

jobs:
  build:
    steps:
      # ...

      - name: Build firmware for Corne RGB
        run: |
          west build -s zmk/app -b nice_nano_v2 -- -DSHIELD=corne_left -DZMK_CONFIG="$(pwd)/config/keyboards/corne_rgb"
          west build -s zmk/app -b nice_nano_v2 -- -DSHIELD=corne_right -DZMK_CONFIG="$(pwd)/config/keyboards/corne_rgb"

      - name: Build firmware for Corne Nice!View
        run: |
          west build -s zmk/app -b nice_nano_v2 -- -DSHIELD="corne_left nice_view_adapter nice_view" -DZMK_CONFIG="$(pwd)/config/keyboards/corne_oled"
          west build -s zmk/app -b nice_nano_v2 -- -DSHIELD="corne_right nice_view_adapter nice_view" -DZMK_CONFIG="$(pwd)/config/keyboards/corne_oled"
Use code with caution.

Summary of the build process
GitHub Actions will run the workflow and build two sets of firmware: one for the Corne RGB and one for the Corne Nice!View.
The keymap is centrally defined in common.dtsi.
Each keyboard's specific overlay and Kconfig files live in their own subfolder under config/keyboards.
The build.yml file points to the correct configuration folder using the -DZMK_CONFIG flag, ensuring each keyboard is built with its unique device features.
After the build completes, you will have two different sets of .uf2 files to flash to the respective keyboards.




