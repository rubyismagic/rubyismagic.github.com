---
layout: post
title: "Episode #7: Closures"
date: 2012-01-19 22:55
comments: true
categories:
---

Herzlich willkommen zu der ~~ersten~~ siebten Ausgabe von 'Ruby is Magic'. Für alle
die am 18.01. bei der [colognerb](http://colognerb.de) nicht live mit
dabei sein konnten, oder einfach noch mal lesen wollen was passiert ist,
kommt hier nun die schriftliche Zusammenfassung mit Codestücken und
Ponies.

Inspiriert von einem hervorragenden [Beitrag von Paul
Cantrell](http://innig.net/software/ruby/closures-in-ruby.html), haben
wir uns in dieser Folge einem Ruby-Thema gewidmet, dem jeder
Ruby-Entwickler regelmäßig begegnet: _Blöcke und Closures_. Wie bei
vielem, dass wir regelmäßig verwenden lohnt sich aber auch hier ein
Blick hinter das Offensichtliche. Und vielleicht entdeckt man etwas,
dass einem bisher so nicht klar war. Wir hoffen also euch neue
Erkenntnisse über Closures in Ruby nahe zu bringen – wir hatten
jedenfalls welche bei den Vorbereitungen der Show.

<!-- more -->

Wie so oft lassen sich Eigenschaften von Programmiersprachen am besten
in der Programmiersprache selbst verdeutlichen. Bevor wir euch aber mit
einem Code-Fragment nach dem anderen bewerfen, wollen wir kurz eine
Definition von Closures aufstellen. Sie erhebt dabei keinerlei Anspruch
auf Vollständigkeit, wir brauchen nur eine gemeinsame Grundlage.

**Wir verstehen Closures als Codeblöcke, die …**

 * zugewiesen und herumgereicht werden können.
 * jederzeit und von jedem aufgerufen werden können.
 * Zugriff auf Variablen im ursprünglich definierenden Kontext haben.

Diese Definition ist nicht exklusiv gültig für Ruby, sondern lässt sich
auch auf andere Sprachen anwenden. In Ruby zeichnen sich Closures
darüber hinaus dadurch aus, dass sie auf die Methode `call()` antworten.
Gleichzeitig ist natürlich nicht jedes Objekt, dass auf `call()`
antworten kann auch ein Closure.

## Die Sieben ist unsere Zahl

{% img right /images/posts/episode_7_closures/sieben.png 400 289 %}

Wirklich bemerkenswert – und durchaus auch verwirrend – ist die Fülle an
Konstrukten, die in Ruby Closures oder Closure-ähnliches beschreiben: Es
gibt derer sieben an der Zahl. Auf den ersten Blick sieht das nach einer
ganzen Menge aus, es stellt sich aber heraus, dass es am Ende dann doch
wieder einfacher ist als gedacht.

Jetzt tun wir aber mal endlich Butter bei die Fische und schauen uns ein
wenig Ruby-Code an. Den Anfang machen einfach _Blöcke_. Solche wie sie
wohl jeder von uns täglich im Zusammenhang mit Iteratoren verwendet:

{% codeblock Einfache Blöcke lang:ruby %}
@my_bag = Bag.new

%w(MacBook Headphones iPhone Camera).each do |item|
  @my_bag.add item
end
{% endcodeblock %}

Der an `each` übergebene Codeblock wird für jedes Element in dem Array
aufgerufen und hat dabei Zugriff auf den umgebenen Kontext. Dadurch
können wir mit `@my_bag` interagieren. Damit ist eines unserer
definierenden Kriterien schon mal erfüllt.

Angenommen wir benötigen nun eine Methode `each_item` auf der Klasse
`Bag` die es uns ermöglicht über alles zu iterieren, was jemand dort
hineingesteckt hat. Eine mögliche Implementierung könnte dann wie folgt
aussehen:

{% codeblock 'each_item'-Implementierung lang:ruby %}
class Bag
  def each_item
    @items.each do |item|
      yield item
    end
  end
end

@my_bag.each_item { |item| puts item }
{% endcodeblock %}

Bemerkenswert an dieser Stelle ist, dass wir in der Methodensignatur
keine Parameter definiert haben. Blöcke können also einfach implizit an
Methoden weitergereicht werden. Gleichzeitig lässt sich der Block
dadurch auch nur implizit über das Schlüsselwort `yield` referenzieren.
Objekte, die man `yield` übergibt werden als Parameter an den Block
weiter gereicht. Falls man `yield` verwendet und keinen Block übergeben
hat, beschwert sich Ruby mit der Meldung `LocalJumpError: no block given (yield)`.
Ob ein Block übergeben wurde oder nicht, lässt sich mit der Methode `block_given?`
prüfen.

Blöcke fangen den definierenden Kontext zwar ein, können ihn jedoch (zum
Glück) nicht erweitern. So erhält man dann auch bei der Ausführung des
nachfolgenden Codes Fehler.

{% codeblock Blöcke erweitern den definierenden Kontext nicht lang:ruby %}
%w(MacBook Headphones iPhone Camera).each do |item|
  item_count ||= 0
  @my_bag.add item
  item_count += 1
end

puts "#{item_count} item(s) have been added to my bag."

# => NameError: undefined local variable or method ‘item_count’
{% endcodeblock %}

Insgesamt sind Blöcke also schon mal ein guter Schritt in die richtige Richtung.
Allerdings fehlen noch zwei Eigenschaften:

 * Wir können den Block nicht beliebig herumreichen (nur einmal beim
 Methodenaufruf) _und_
 * Wir können den Block nicht jederzeit aufrufen.

Beiden Problemen lässt sich damit begegnen, in dem man den Block
explizit macht, also in die Methodensignatur aufnimmt:

{% codeblock Expliziter Block lang:ruby %}
class Bag
  def initialize(items)
    @items = items
  end

  def each_item(&block)
    @items.each(&block)
  end
end

bag = Bag.new %w(MacBook Headphones Keys)
bag.each_item { |item| puts item }
{% endcodeblock %}

Als Nutzer der Methode ändert sich für mich dadurch übrigens überhaupt
nichts. Ich erhalte nicht einmal einen `ArgumentError` wenn ich den
Block nicht an die Methode übergebe – selbst nicht bei dem Aufruf von
`@items.each(&block)`, denn `each` liefert einfach eine
`Enumerator`-Instanz zurück, wenn kein Block mitgegeben wird.

Als letzten Schritt müssen wir nun nur noch dafür sorgen, dass der Block
gespeichert werden kann und somit jederzeit aufrufbar wird. Wir
erreichen das, indem wir uns einfach das `&`-Prefix des Blockparameters
sparen, was ein Synonym für `Proc.new(&block)` ist.

{% codeblock Speichern des Blocks lang:ruby %}
class Bag
  def initialize(items)
    @items = items
  end

  def define_iterator(&block)
    @iterator = block # Proc.new &block
  end

  def iterate!
    @items.each(&@iterator)
  end
end

bag = Bag.new(%w(MacBook Headphones Keys))
bag.define_iterator { |item| puts item }
bag.iterate!
{% endcodeblock %}

Damit haben wir also endlich unser Closure. Wir hoffen es hat euch mal
wieder gefallen und bis zum nächsten Mal … Moment! … Das kann es doch
noch nicht gewesen sein. Richtig – da kommt noch was!

## "Echte" Closures?!

Wie Eingangs bereits erwähnt, hält Ruby für uns mehrere Möglichkeiten
bereit ein Closure zu konstruieren:

 * `&block` ohne `&` ist wie `Proc.new(&block)`
 * `proc {}`
 * `lambda {}`

Die Variante mit `proc` ist übrigens noch keinem von uns bewusst in der
freien Wildbahn aufgefallen und es handelt sich dabei auch lediglich um
ein Alias auf `lambda`. Was irgendwie nicht besonders intuitiv ist. Das
hat sich dann wohl auch das Ruby-Core-Team gedacht und ab Ruby 1.9 ist
`proc` sinniger Weise ein Alias auf `Proc.new`.

Dennoch stellt sich die Frage, ob Unterschiede zwischen den `lambda` und
`Proc.new` Varianten existieren? Und wenn ja, welche? Die Antwort ist zu
erwarten gewesen: Ja, es gibt Unterschiede: Auswirkungen auf den _Kontrollfluss_
und die _Prüfung der Arität_.

### Kontrollfluss

Wenn man innerhalb eines Closures oder Code-Blocks `return` verwenden
möchte führt dies unter Umständen nicht immer zu dem gewünschten
Resultat:

{% codeblock Unterschiedliches Verhalten bei 'return' lang:ruby %}
def call_closure(closure)
  puts "Calling a closure"
  result = closure.call
  puts "The result of the call was: #{result}"
end

call_closure(Proc.new { return "All hell breaks loose!" })

# => Calling a closure
# => LocalJumpError: unexpected return

call_closure(lambda { return "Everypony calm down. All is good." })

# => Calling a closure
# => The result of the call was: ‘Everypony calm down. All is good.’
{% endcodeblock %}

Während sich also ein `return` in einem durch `Proc.new` erzeugten
Closure immer auf den ursprünglich definierenden Kontext bezieht,
springt es in einem `lambda`-Closure einfach aus der Funktion zurück.
Bei der Verwendung der `lambda`-Methode erhält man also eine "true
closure". In beiden Fällen erhält man übrigens eine Instanz der
`Proc`-Klasse.

### Aritätsprüfung

### Fun Facts

{% img left /images/ponies/pinkie_pie.png 280 311 %}

## One More Thing

## Präsentation

Und hier noch die Präsentation. Für das volle audio-visuelle Erlebnis
müsst ihr allerdings zur [@colognerb](http://twitter.com/colognerb) kommen ;-)

<div style="width:595px;margin:auto" id="__ss_11190532"> <object id="__sse11190532" width="595" height="497"> <param name="movie" value="http://static.slidesharecdn.com/swf/ssplayer2.swf?doc=colognerb-closures-120121083720-phpapp02&rel=0&stripped_title=ruby-is-magic-episode-7-closures&userName=railsbros_dirk" /> <param name="allowFullScreen" value="true"/> <param name="allowScriptAccess" value="always"/> <param name="wmode" value="transparent"/> <embed name="__sse11190532" src="http://static.slidesharecdn.com/swf/ssplayer2.swf?doc=colognerb-closures-120121083720-phpapp02&rel=0&stripped_title=ruby-is-magic-episode-7-closures&userName=railsbros_dirk" type="application/x-shockwave-flash" allowscriptaccess="always" allowfullscreen="true" wmode="transparent" width="595" height="497"></embed> </object> </div>
