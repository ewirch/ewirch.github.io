Site Generator: Jekyll jekyllrb.com
Erfordert: Ruby, RubyGems (https://jekyllrb.com/docs/installation/#requirements)
GitHub erkennt automatisch Jekyll-Dateien und generiert die Seite selber.
Vorlage: https://github.com/mmistakes/skinny-bones-jekyll/

Seite bauen: bundle exec jekyll build
Lokalen http-Server starten: bundle exec jekyll serve  --config _config.yml,_config-dev.yml --drafts
Die Config-Parameter sind dazu da, dass die Links auf localhost zeigen.

Immer wenn sich Dependencies ändern:
bundle update
bundle install

