Site Generator: Jekyll jekyllrb.com
Erfordert: Ruby, RubyGems (https://jekyllrb.com/docs/installation/#requirements)
GitHub erkennt automatisch Jekyll-Dateien und generiert die Seite selber.
Vorlage: https://github.com/mmistakes/skinny-bones-jekyll/

Seite bauen: bundle exec jekyll build
Lokalen http-Server starten:
 bundle exec jekyll serve  --config _config.yml,_config-dev.yml --drafts
Die Config-Parameter sind dazu da, dass die Links auf localhost zeigen.

Immer wenn sich Dependencies ändern:
bundle update
bundle install

Links validieren:
bundle exec htmlproofer ./_site/ --only-4xx

Erweiterte Markdown-Syntax
==========================

Fußnoten
--------
Dies ist ein Fließtext[^footnote-id].

[^footnote-id] Dies ist der Fußnoten-Text, mitten zwischen zwei Absätzen. Gerendert wird er aber zum Schluss des Artikels.

Links
-----
Hier ist ein [Link][].

Am Ende des Dokuments:
[Link]: https://meinlink.de

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

