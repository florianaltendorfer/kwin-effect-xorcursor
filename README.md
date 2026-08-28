# kwin-effect-xorcursor

XOR cursor effect for KDE Plasma

![IMG_20250527_210735](https://github.com/user-attachments/assets/3a3eeba0-4e56-4925-897c-e3607c03f8bf)

## Note

Use a white cursor theme for the best effect.

## Build

```bash
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
cmake --build build
sudo cmake --install build
```

## Changes required to turn this into a proper implementation

1. Correctness: Track mouse show/hide states. Other effects like `shake cursor to enlarge` or `auto hide cursor` might also hide the cursor and / or draw their own. This effect should somehow avoid conflicting with them.
2. Performance:
2.1. Don't request a fullscreen repaint on mouse movement. Repaint only the old and new mouse positions.
2.2. Only paint the mouse when it is visible in the repaint region.
