Source for Hugo (gohugo.io) based blog.

To add the theme:
```
$ mkdir themes
$ cd themes
$ git clone https://github.com/halogenica/beautifulhugo.git beautifulhugo
```

Make this change in the theme layouts/_default/single.html
```html
          <div class="disqus-comments">
            <!--{{ template "_internal/disqus.html" . }}-->
            {{ partial "disqus.html" . }}
          </div>
```
