
Erfordert:
  - Ruby, RubyGems (https://jekyllrb.com/docs/installation/#requirements)
  - ruby-bundler
  - Site Generator: Jekyll jekyllrb.com

GitHub erkennt automatisch Jekyll-Dateien und generiert die Seite selber.
Vorlage: https://github.com/mmistakes/minimal-mistakes

Seite bauen: bundle exec jekyll build
Lokalen http-Server starten:
 bundle exec jekyll serve  --config _config.yml,_config-dev.yml --drafts
Die Config-Parameter sind dazu da, dass die Links auf localhost zeigen.

Immer wenn sich Dependencies ändern:
bundle update
bundle install

Links validieren:
 - 'gem "html-proofer"' temporär zu Gemfile hinzufügen
 - bundle install
 - bundle exec htmlproofer ./_site --check-html --disable-external


Erweiterte Markdown-Syntax
==========================

Fußnoten
--------
Dies ist ein Fließtext[^footnote-id].

[^footnote-id] Dies ist der Fußnoten-Text, mitten zwischen zwei Absätzen. Gerendert wird er aber zum Schluss des Artikels.

Links
-----
Hier ist ein [Link][]. Hier ist ein [Link mit separater ID][link-id].

Am Ende des Dokuments:
[Link]: https://meinlink.de
[link-id]: https://meinlink.de

Akronymerklärung
------------------
*[vsyncs]: Am Ende des Dokuments, unterstreicht jedes Vorkomnis von "vsyncs" im Fließtext und blendet diesen Text ein.

Begriffsdefinition
------------------
*Ein Begriff*
: Erklärt durch Fließtext.

Interne Verlinkung mit Link Validierung
---------------------------------------
{{site.url}}{% post_url 2017-09-09-better-commits-3-review-changes %}

