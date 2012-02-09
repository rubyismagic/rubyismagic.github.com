---
layout: post
title: "Behind the Scenes: Decorators"
date: 2012-02-09 13:27
comments: true
categories: closures pattern
---

Willkommen bei "Ruby is Magic – Behind the Scenes". Wenn ihr euch noch
an die [letzte Episode](blog/2012/01/19/episode-7-closures/) erinnert, dann haben wir gezeigt, wie sich in Ruby
Methoden als Closuren verwenden lassen. Dazu haben wir das
[Decorator-Pattern](http://en.wikipedia.org/wiki/Decorator_pattern) ähnlich [wie in Python](http://stackoverflow.com/questions/101268/hidden-features-of-python#101447) [implementiert](https://gist.github.com/294f56ed664efa99dcac).

Das Transkript der letzten Show war allerdings schon recht lang und
daher sind wir nicht näher auf die Implementierung eingegangen. Da sie
jedoch sehr interessant ist, wollen wir in diesem Artikel noch einmal
im Detail darauf eingehen. Als kleinen Bonus haben wir das ganze auch
einmal einem Benchmark unterzogen – Natürlich völlig
nicht-repräsentativ ;-)

<!-- more -->

Zunächst aber noch einmal kurz zum Hintergrund: Es ging vor
allem darum einen halbwegs realen Anwendungsfall für die Verwendung von
`method()` zu finden. Das was am Ende dann hinten raus gefallen ist,
lässt sich unserer Meinung nach sogar tatsächlich in realen
Projekten einsetzen, denn das Decorator-Pattern ermöglicht durchaus elegante
Lösungen, wenn man eine Reihe unterschiedlicher Methoden mit
zusätzlichen Funktionalitäten anreichern möchte. In unserem Beispiel war
es dann eben das Hinzufügen einer Caching-Schicht.

Nehmen wir als Ausgangspunkt einmal ein stark vereinfachtes Objekt, um
auf eine relationale Datenbank zuzugreifen:

{% codeblock Datenbank Anbindung ohne Caching lang:ruby %}
class DatabaseConnector
  def find(id)
    @connection.perform_sql("SELECT * FROM ? WHERE id = ?", self.table_name, id)
  end
end
{% endcodeblock %}

Wir haben hier, wie so oft, unterschiedliche Möglichkeiten Caching
hinzuzufügen. Eine Variante wäre die Verwendung von `alias_method` oder
eben `alias_method_chain`, wenn ActiveSupport mit von der Partie ist. Das Ergebnis
ist, dass man eine weitere Methode in seiner Klasse definiert, die
das Caching implementiert. Dann stellen wir uns vor, dass wir noch zwei
bis drei weitere Methoden cachen wollen. Die einzelnen Implementierungen
werden dabei recht ähnlich zueinander sein und zusätzlich ist die Klasse
überladen mit Methoden, die nicht wirklich in ihren Aufgabenbereich
fallen:

{% codeblock Datenbank Anbindung mit Caching lang:ruby %}
class DatabaseConnector
  def find(id)
    @connection.perform_sql("SELECT * FROM ? WHERE id = ?", self.table_name, id)
  end

  def find_with_cache(id)
    MyCache.instance.fetch("#{self.table_name}-find-#{id}") { find_without_cache(id) }
  end
  alias_method_chain :find, :cache

  def count
    @connection.perform_sql("SELECT COUNT(*) FROM ?", self.table_name)
  end

  def count_with_cache
    MyCache.instance.fetch("#{self.table_name}-count") { count_without_cache }
  end
  alias_method_chain :count, :cache
end
{% endcodeblock %}

Das erste Problem lässt sich lösen, indem die Cache-Funktionalität in
einer eigenen Klasse weiter gekapselt und diese in jeder Methode
aufgerufen wird. Realisiert man diese Klasse nun noch als Decorator,
haben wir auch das zweite Problem aus der Welt geschafft:

{% codeblock Datenbank Anbindung mit Decorator lang:ruby %}
class CacheDecorator
  def call(*params)
    MyCache.instance.fetch(cache_key(*params)) {
      @component.call(*params)
    }
  end
end

class DatabaseConnector
  decorate CacheDecorator
  def find(id)
    @connection.perform_sql("SELECT * FROM ? WHERE id = ?", self.table_name, id)
  end
end
{% endcodeblock %}

Die Konstruktion des Cache-Keys ist natürlich jetzt etwas komplexer
geworden, aber irgendwas ist ja immer …

## Implementierung

Kommen wir nun langsam zum interessanten Teil: Die Implementierung.
Vergleicht man die `alias_method_chain`-Methode mit der
Decorator-Variante, dann fallen zwei Unterschiede auf:

 1. Die Methode `decorate` wird _vor_ der zu dekorierenden Methode
 aufgerufen.
 2. Der Name der zu dekorierenden Methode wird nicht angegeben –
 lediglich die Decorator-Klasse.

Überlegen wir uns zunächst einmal welche Schritte notwendig sein
werden, um unsere Decorator Funktionalität umzusetzen:

 1. Erkennen welche Methode zu dekorieren ist
 2. Methode extrahieren – `method()`
 3. Decorator mit extrahierter Methode inititalisieren
 4. Proxy Methode definieren
 5. Binding vor Ausführung der “alten” Methode umsetzen

Im Kontext von Closures sind nur die Punkte 2. und 5. von Relevanz. Der
Rest ist im Grunde nur Glue-Code – Was ihn jedoch nicht weniger
interessant macht.

Das ganze in Ruby-Code gegossen ergibt dann nicht einmal 40 Zeilen
– wieder einmal ein schönes Beispiel dafür, wie ausdrucksstark diese
Sprache ist:

{% codeblock Implementierung des Decorator-Pattern lang:ruby %}
module FunctionDecorators
  def decorate(decorator)
    @decorate_next_with = decorator
  end

  def method_added(name)
    if decorator_class = @decorate_next_with
      @decorate_next_with = nil
      apply_decorator(decorator_class, name, self)
    end
  end

  def __decorators
    @_decorators ||= {}
  end

  def apply_decorator(decorator, method_name, target)
    decorated_method = target.instance_method(method_name)

    target.send(:remove_method, method_name)

    target.__decorators[method_name] = { :decorator => decorator.new, :method => decorated_method }

    new_method = <<-RUBY
      def #{method_name}(*params, &block)
        unless @_#{method_name}_decorator
          @_#{method_name}_decorator, decorated_method = self.class.__decorators[:#{method_name}].values
          @_#{method_name}_decorator.bind(decorated_method, self)
        end

        @_#{method_name}_decorator.call(*params, &block)
      end
    RUBY

    target.class_eval new_method
  end
end
{% endcodeblock %}

Die angesprochenen Unterschiede zu `alias_method` haben wir durch einen
der zahlreichen (und teilweise nicht dokumentierten) Callbacks in Ruby
realisiert: Der Methodenaufruf `decorate` merkt sich einfach die
Decorator-Klasse und sobald die nächste Methode definiert wird, wird
diese damit dekoriert. Auf das Hinzufügen einer Methode lässt sich dann mit
dem Callback `method_added` warten.

Im Callback wird dann die zu dekorierende Methode extrahiert und durch
eine neue ersetzt. Die neue Methode braucht dabei nicht die gleiche
Signatur zu besitzen (das hatten wir in der letzten Episode noch
anders) – alle Parameter einsammeln und einen optionalen Block-Parameter
definieren reicht schon aus. Auf Klassenebene legen wir unter dem
Methodennamen noch die ursprüngliche Methode und eine Instanz der
Decorator-Klasse ab. Das ist notwendig, weil wir eine `UnboundMethod`
extrahieren, d.h. sie ist keinem Objekt zugeordnet und lässt sich damit
auch nicht aufrufen. Das `bind` wird dann durchgeführt sobald die
neu-definierte Methode auf einer konkreten Instanz aufgerufen wird.
Damit erhalten wir eine Instanz von `Method`, die sich wie ein Closure
aufrufen lässt.

Das ganze ist als Modul realisert, welches per `extend` entweder direkt
in `Object` oder etwas selektiver nur in die zu dekorierenden Klassen
eingebunden werden kann.

Zur Vollständigkeit hier noch die komplette Implementierung des
`CacheDecorator`. Erwähnenswert ist noch, dass man über die `receiver`
auf der ursprünglichen Methode `@component` die die Instanz des
dekorierten Objekts erhält. Somit hat man schon hier Zugriff auf den
Kontext von `@component` und kann zum Beispiel den Cache-Key
konstruieren:

{% codeblock Implementierung des Decorators lang:ruby %}
class CacheDecorator
  def call(*params)
    MyCache.instance.fetch(cache_key(*params)) {
      @component.call(*params)
    }
  end

  def cache_key(*params)
    "#{@component.receiver.table_name}-#{@component.name}"
  end

  def bind(component, receiver)
    @component = component.bind(receiver)
  end
end
{% endcodeblock %}

## Benchmark

Wie versprochen gibt es am Ende noch einen nicht-repräsentativen
Benchmark für die Verwendung unserer Decorator-Implementierung. Wir
haben eine alternative Implementierung mit `alias_method_chain`, also
unter Verwendung von ActiveSupport mit dem Decorator-Ansatz verglichen.
Konkret sah die Teststellung wie folgt aus:

{% codeblock Benchmark lang:ruby %}
runs = 1_000_000

Benchmark.bm(40) do |x|
  x.report("Cache with alias") do
    runs.times do
      connector = DatabaseConnectorWithAlias.new
      connector.find(42)
      connector.count
    end
  end

  x.report("Cache with alias (single instance)") do
    connector = DatabaseConnectorWithAlias.new
    runs.times do
      connector.find(42)
      connector.count
    end
  end

  x.report("Cache with Decorator") do
    runs.times do
      connector = DatabaseConnectorWithDecorator.new
      connector.find(23)
      connector.count
    end
  end

  x.report("Cache with Decorator (single instance)") do
    connector = DatabaseConnectorWithDecorator.new
    runs.times do
      connector.find(23)
      connector.count
    end
  end
end
{% endcodeblock %}

Wie erwartet ist die Decorator-Implementierung signifikant langsamer
als `alias_method_chain`:

{% codeblock %}
______                                         user     system      total        real
Cache with alias                           2.650000   0.010000   2.660000 (  2.671713)
Cache with alias (single instance)         2.370000   0.000000   2.370000 (  2.419900)
Cache with Decorator                      13.620000   0.020000  13.640000 ( 13.794589)
Cache with Decorator (single instance)     8.750000   0.020000   8.770000 (  8.809148)
{% endcodeblock %}

Aber es war ja auch nicht das Ziel `alias_method_chain` zu ersetzen,
sondern einen Anwendungsfall für Methoden als Closures zu finden. Wir
denken, dass ist uns gut gelungen und wir werden diese Implementierung
definitiv irgendwo mal verwenden, denn es macht den Code durchaus
übersichtlicher und die Decorator besser testbar.

Das war es auch schon mit dem ersten "Behind the Scenes". Wir hoffen es
hat auch gefallen. Wir sehen uns bei der nächsten "Ruby is Magic"-Show
am 15.02.2012 auf der [cologne.rb](http://colognerb.de).
