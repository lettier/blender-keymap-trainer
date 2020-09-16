<p align="center">
<img src="https://i.imgur.com/Ocu1usp.png" alt="Blender Keymap Trainer">
</p>

# Blender Keymap Trainer

A digital flashcards application for memorizing Blender's shortcut keys or keymap.
Click [here](https://lettier.github.io/blender-keymap-trainer) to use Blender Keymap Trainer.

## Features

<p align="center">
<img src="https://i.imgur.com/O9AnQdy.gif">
</p>

- It remembers your progress and settings.
- You can upload your own keymap.
- Filter the available cards you wish to memorize.
- The cards you miss are more likely to be chosen next.
- You can switch between the key or action side.
- Skip the cards you're not interested in.

## Uploading Your Own Keymap

<p align="center">
<img src="https://i.imgur.com/Bd2Eu2x.png">
</p>

From Blender, select `Edit > Preferences > Keymap > Export`.

Open the exported Python file and replace (located at the bottom)

```python
if __name__ == "__main__":
  import os
  from bl_keymap_utils.io import keyconfig_import_from_data
  keyconfig_import_from_data(os.path.splitext(os.path.basename(__file__))[0], keyconfig_data)
```

with the following:

```python
if __name__ == "__main__":
  import json
  with open('./keymap.json', 'w') as file:
    file.write(json.dumps(keyconfig_data))
```

Now run the file with `python exported-file.py`.

<p align="center">
<img src="https://i.imgur.com/etmuIy3.png">
</p>

You can now upload the JSON file from Blender Keymap Trainer.

## Filtering

<p align="center">
<img src="https://i.imgur.com/8eX3Jig.gif">
</p>

You can filter by key/action, section, space, or region.
This allows you to focus only on the node editor keys for example.

## Copyright

(C) 2020 David Lettier
<br>
[lettier.com](https://www.lettier.com)
