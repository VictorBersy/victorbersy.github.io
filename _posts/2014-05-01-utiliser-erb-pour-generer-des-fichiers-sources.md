---
layout: post
title: Utiliser ERB pour générer des fichiers sources
categories: Code
author: Victor Bersy
---
ERB est bien connu dans le monde de Ruby / Ruby On Rails.
ERB signifie "Embedded RuBy", ou en Français RBE pour "RuBy Embarqué". C'est un système de template, utilisé notamment par Rails afin de générer les fichiers HTML. Il est similaire aux autres systèmes de templates comme par exemple [Twig](http://twig.sensiolabs.org/) en PHP.

Cependant, ce que vous ne savez peut être pas, c'est qu'on peut aussi l'utiliser pour générer des fichiers autre que du HTML.

C'est ce que j'ai fait pour créer le générateurs d'extensions qui animent mon projet en cours, [Jarvis](https://github.com/Jarvis-Bot/Jarvis-Core).
Pour faciliter la vie des développeurs de Jarvis, je me suis dit qu'un générateur des fichiers nécessaires à chaque extension serait plus facile à utiliser que de devoir fouiner dans la [documentation de Jarvis](https://jarvis-bot.github.io/).

1. Récolter les données
-----------------------

Avant de générer ces fichiers, il faut d'abord récolter les données nécessaires. J'ai donc décidé de créer une interface en ligne de commandes, qui soit à la fois simple et intuitive.
Basée sur les travaux de [Ninefold](https://github.com/ninefold/cli/), particulièrement sur ce fichier [stdio.rb](https://github.com/ninefold/cli/blob/master/lib/ninefold/services/stdio.rb), je l'ai modifiée à ma guise et voici le rendu :

![Capture d'écran générateur sources]({{ site.url }}/assets/posts/2014-05-01-utiliser-erb-pour-generer-des-fichiers-sources/1.png)

Dans cet article, on va se contenter de quelque chose de plus basique, dans une approche pas très ruby.

{% highlight ruby %}
class AskData
  def initialize
    @answers = {}
  end

  def ask_questions
    print "What's the name of your plugin? "
    @answers['plugin_name'] = $stdin.gets.chomp.strip
    print "What's the class name of your plugin? "
    @answers['class_name'] = $stdin.gets.chomp.strip
  end

  def answers
    @answers
  end
end
{% endhighlight %}

Cette classe s'occupera de récolter les données nécessaires à la création de notre fichier source.

2. Créer les modèles
--------------------

Pour pouvoir générer des fichiers, ERB doit disposer d'un fichier sur lequel se baser. Appelé modèle, ou template, c'est un fichier qui contient des variables. Voici un exemple simple de modèle possible :

{% highlight erb %}
Bienvenue chez <%= @infos['company']['name'].capitalize %>, <%= @infos['user']['pseudo'].capitalize %> !
{% endhighlight %}

Dans ce court exemple, on récupère les informations fournies par ERB pour les injecter dans le modèle et générer le fichier. Voici le rendu possible :

    Bienvenue chez Very Wow Entreprise, SuchDoge_0148 !

Voici le modèle utilisé ici :

{% highlight erb %}
class <%= @data['class_name'].capitalize %>
  def initialize
  	# This method will be called by Jarvis on boot.
  end

  def create_message
  	# To communicate with Jarvis, you've to send him a message like this :
  	# Jarvis::Messages::Message.new('<%= @data['plugin_name'] %>', 'Hello Jarvis!', object)
  end
end
{% endhighlight %}


Je vous conseille de placer les modèles dans un dossier séparé, respectant la norme suivante : `nom_fichier_final.extension_final.erb`. Ainsi pour générer `addon_init.rb` le modèle s'appellera `addon_init.rb.erb`.


3. Générateur
-------------

Une fois le modèle créé et les données récoltées, il faut maintenant créer notre générateur de fichier.
Il sera nécessaire d'utiliser erb. Par conséquent, n'oubliez pas d'insérer `require 'erb'` au début du document.

{% highlight ruby %}
require 'erb'
class FilesGenerator
end
{% endhighlight %}

ERB ne reçoit pas les données en paramètres, mais grâce à un `binding`, une liaison automatique entre la classe mère et le template. Dans votre classe `FilesGenerator`, vous devez recevoir vos données, et les rendre accessible à l'extérieur de la classe via `attr_accessor`.

{% highlight ruby %}
require 'erb'
class FilesGenerator
  attr_accessor :data
  def initialize(data)
    @data = data
  end
end
{% endhighlight %}


Pour plus de souplesse dans notre classe, on va pouvoir passer en argument le nom du fichier à générer (`file_name`) et le chemin dans lequel on veut le générer (`path`).

{% highlight ruby %}
def initialize(file_name, path, data)
  @file_name = file_name
  @path = path
  @data = data
  @template = File.read(File.join(__dir__, 'templates', "#{file_name}.erb"))
end
{% endhighlight %}


Pour générer nos fichiers, on va créer la méthode `generate`.

{% highlight ruby %}
def generate
  @rendered = ERB.new(@template).result(binding)
end
{% endhighlight %}

Remarquez le `binding` expliqué plus tôt, passé en paramètre à la méthode `result` de ERB.

Enfin, la méthode `write`, qui va écrire nos fichiers.

{% highlight ruby %}
def write
  File.open(File.join(@path, @file_name), 'w') do |f|
    f.write(@rendered)
  end
end
{% endhighlight %}


Voici la classe finale :

{% highlight ruby %}
require 'erb'
class FilesGenerator
  attr_accessor :data
  def initialize(file_name, path, data)
    @file_name = file_name
    @path = path
    @data = data
    @template = File.read(File.join(__dir__, 'templates', "#{file_name}.erb"))
  end

  def generate
    @rendered = ERB.new(@template).result(binding)
  end

  def write
    File.open(File.join(@path, @file_name), 'w') do |f|
      f.write(@rendered)
    end
  end
end
{% endhighlight %}


4. Assemblage
-------------
Il suffit maintenant d'assembler tout ça dans un seul fichier :
{% highlight ruby %}
require './ask_data'
require './files_generator'

asker = AskData.new
data = asker.ask_questions
data = asker.answers

generator = FilesGenerator.new('addon_init.rb', 'rendered', data)
generator.generate
generator.write
{% endhighlight %}


Conclusion
----------

Et voilà, vous pouvez désormais utiliser ces techniques pour générer vos propres fichiers. Que ce soient des fichiers en Ruby, ou dans un autre langage, des données provenant d'un API, d'un utilisateur ... Les possibilités sont infinies !

Vous pouvez retrouver les fichiers utilisés dans cet article sur [github.com/VictorBersy/tutorials](https://github.com/VictorBersy/tutorials/tree/master/1_G%C3%A9n%C3%A9rer-fichiers-avec-erb).