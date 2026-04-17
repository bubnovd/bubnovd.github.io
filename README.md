`docker run -it --rm --platform=linux/amd64 -v $(pwd):/src -p 1313:1313 peaceiris/hugo:v0.146.4-full serve -D --bind 0.0.0.0`

`themes/hugo-theme-cleanwhite/layouts/partials/footer.html:314`:
```go
{{ template "_internal/google_analytics.html" . }}
```

17.04.2026
Перезаписал layout/footer сабмодуля на собвтенный, чтобы избежать ошибок google analytics 
