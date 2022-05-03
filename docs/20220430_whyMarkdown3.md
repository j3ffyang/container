---

presentation:
  # presentation theme
  # === available themes ===
  # "beige.css"
  # "black.css"
  # "blood.css"
  # "league.css"
  # "moon.css"
  # "night.css"
  # "serif.css"
  # "simple.css"
  # "sky.css"
  # "solarized.css"
  # "white.css"
  # "none.css"
  theme: black.css

  # The "normal" size of the presentation, aspect ratio will be preserved
  # when the presentation is scaled to fit different resolutions. Can be
  # specified using percentage units.
  width: 960
  height: 700

  # Factor of the display size that should remain empty around the content
  margin: 0.1

  # Bounds for smallest/largest possible scale to apply to content
  minScale: 0.2
  maxScale: 1.5

  # Display controls in the bottom right corner
  controls: true

  # Display a presentation progress bar
  progress: true

  # Display the page number of the current slide
  slideNumber: true

  # Push each slide change to the browser history
  history: false

  # Enable keyboard shortcuts for navigation
  keyboard: true

  # Enable the slide overview mode
  overview: true

  # Vertical centering of slides
  center: true

  # Enables touch navigation on devices with touch input
  touch: true

  # Loop the presentation
  loop: false

  # Change the presentation direction to be RTL
  rtl: false

  # Randomizes the order of slides each time the presentation loads
  shuffle: false

  # Turns fragments on and off globally
  fragments: true

  # Flags if the presentation is running in an embedded mode,
  # i.e. contained within a limited portion of the screen
  embedded: false

  # Flags if we should show a help overlay when the questionmark
  # key is pressed
  help: true

  # Flags if speaker notes should be visible to all viewers
  showNotes: false

  # Number of milliseconds between automatically proceeding to the
  # next slide, disabled when set to 0, this value can be overwritten
  # by using a data-autoslide attribute on your slides
  autoSlide: 0

  # Stop auto-sliding after user input
  autoSlideStoppable: true

  # Enable slide navigation via mouse wheel
  mouseWheel: false

  # Hides the address bar on mobile devices
  hideAddressBar: true

  # Opens links in an iframe preview overlay
  previewLinks: false

  # Transition style
  transition: 'default' # none/fade/slide/convex/concave/zoom

  # Transition speed
  transitionSpeed: 'default' # default/fast/slow

  # Transition style for full page slide backgrounds
  backgroundTransition: 'default' # none/fade/slide/convex/concave/zoom

  # Number of slides away from the current that are visible
  viewDistance: 3

  # Parallax background image
  # parallaxBackgroundImage: 'https://s3.amazonaws.com/hakim-static/reveal-js/reveal-parallax-1.jpg' # e.g. "https://s3.amazonaws.com/hakim-static/reveal-js/reveal-parallax-1.jpg"

  # Parallax background size
  parallaxBackgroundSize: '' # CSS syntax, e.g. "2100px 900px" - currently only pixels are supported (don't use % or auto)

  # Number of pixels to move the parallax background per slide
  # - Calculated automatically unless specified
  # - Set to 0 to disable movement along an axis
  parallaxBackgroundHorizontal: 200
  parallaxBackgroundVertical: 50

  # Enable Speaker Notes
  enableSpeakerNotes: false

---

<!-- slide -->
# Markdown

<!-- slide -->
__Markdown__ is a _lightweight markup language_ for creating formatted text using a plain-text editor. John Gruber and Aaron Swartz created Markdown in 2004 as a markup language that is appealing to human readers in its source code form.[9] Markdown is widely used in blogging, instant messaging, online forums, collaborative software, documentation pages, and readme files.

<font size=3>Reference > https://en.wikipedia.org/wiki/Markdown</font>

<!-- slide -->

## Editor

- atom.io (推荐)
- sublimetext.com
- https://code.visualstudio.com/
- ...

<!-- slide -->

## Atom.io Plugin

- `Edit` > `Preferences` > `Install`
- 推荐安装
  - `markdown-preview-enhanced` (所见即所得)
  - `markdown-img-paste`  (直接把截图拖入)

<!-- slide -->

## Document, Presentation and Diagram

- 语言
- Table of Content (ToC)
- Doc
- Presentation
- Table
- Font
- Diagram (plantuml.com | draw.io)
- ...

<!-- slide -->

## Doc > Table of Content

`<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->`

- 自动生成
- `depthFrom=2`: 从2个`#`开始
- `depthTo=4`: 忽略4个`#`以外

<!-- slide -->

## Diagram > Plantuml.com

- 甘特图 > https://plantuml.com/gantt-diagram
- 脑图  > https://plantuml.com/mindmap-diagram
- Configuration >
<font size=5>https://github.com/j3ffyang/container/blob/master/docs/20200317_plantuml.md</font>


<!-- slide -->

## Diagram > Draw.io

- 输出 XML
- 可以在浏览器直接编辑

<!-- slide -->

## Integrate with Github.com

<!-- slide -->

## Print

- PDF
- Browser
<!-- slide -->

## 小结

- 纯文本：编辑，分享，储存
- 方便Git代码审核使用 `diff`
<!-- slide -->

## Reference

- 基本语法 > https://markdown.com.cn/cheat-sheet.html#%E6%80%BB%E8%A7%88
