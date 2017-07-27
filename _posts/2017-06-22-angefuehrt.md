---
layout: page-fullwidth
title: Angeführt
published: true
author: mwerner
meta_description: "Jekyll kann typographische Anführungszeichen."
date: 2017-06-22
tags:
    - jekyll
    - kramdown
categories:
    - small hacks
---
Jekyll -- oder genauer der dort genutzte Markdown-Prozessor Kramdown -- kann typographische Anführungszeichen.
Wenn man also ein Zitat in Anführungszeichen (\") setzt stehen nach der Verarbeitung vor und nach dem Zitat typographische
Anführungszeichen:

   * `"Zitat"` wird zu &ldquo;Zitat&rdquo;

Leider entspricht das nicht der deutschen Typographie, wo vor dem Zitat das Zitatzeichen unten und danach oben (und anders herum) steht.
Durch eine zusätzliche Konfiguration kann man auf deutsche Typographie umschalten, indem man die <kbd>_config.yml</kbd> um folgende Zeilen ergänzt:

~~~ yml
kramdown:
  smart_quotes:   sbquo,lsquo,bdquo,ldquo
~~~
Nun ergibt sich der richtige Schriftsatz im Deutschen:

   * `"Zitat"` wird zu "Zitat"
