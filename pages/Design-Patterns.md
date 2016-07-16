---
layout: page
title: Dizajn paterni (Design patterns)
sitemap: true
---

# Dizajn paterni (Design patterns)

Postoje brojni pristupi i različiti načini za izgradnju kôda vaše web aplikacije, samo je pitanje koliko ćete vremena posvetiti arhitekturi vašeg sistema.
Obično je dobra ideja da se vodite nekim uobičajenim šablonama i praksama, zato što će na taj način vaš kôd biti jednostavniji za održavanje i lakši za razumijevanje.

* [Arhitekturalni patern (Wikipedia)](https://en.wikipedia.org/wiki/Architectural_pattern)
* [Dizajn patern (Wikipedia)](https://en.wikipedia.org/wiki/Software_design_pattern)
* [Primjeri implementacija dizajn paterna](https://github.com/domnikl/DesignPatternsPHP)

## Factory

Jedan od najčešće korištenih dizajn paterna je Factory. U njegovom slučaju, sve što neka klasa radi jeste kreiranje
objekta kojeg namjeravate koristiti. Pogledajmo sljedeći primjer Factory paterna:

{% highlight php %}
<?php
class Automobile
{
    private $vehicleMake;
    private $vehicleModel;

    public function __construct($make, $model)
    {
        $this->vehicleMake = $make;
        $this->vehicleModel = $model;
    }

    public function getMakeAndModel()
    {
        return $this->vehicleMake . ' ' . $this->vehicleModel;
    }
}

class AutomobileFactory
{
    public static function create($make, $model)
    {
        return new Automobile($make, $model);
    }
}

// Factory kreira Automobile objekt
$veyron = AutomobileFactory::create('Bugatti', 'Veyron');

print_r($veyron->getMakeAndModel()); //ispisuje "Bugatti Veyron"
{% endhighlight %}

Ovaj kôd koristi Factory za kreiranje instance Automobile klase. Postoje dva moguća benefita pisanja kôda
na ovaj način. Prvi je to što ćete u slučaju izmjene, preimenovanja ili zamjene Automobile klase, morati 
mijenjati jedino kôd u njenom factory-u, umjesto na svakom mjestu gdje se koristi klasa Automobile. Druga prednost
je ta što u slučaju da je sâmo kreiranje objekta kompleksna operacija, sav taj posao možete obaviti u factory
klasi, umjesto da ga ponavljate svaki put kada vam treba nova instanca.

Korištenje Factory paterna nije uvijek neophodno (ispravno). Prethodni primjer je toliko jednostavan da bi korištenje
factory-a bio samo nepotrebna komplikacija. Ipak, ako radite na velikom i kompleksnom projektu, korištenje
factory-a vas možete lišiti dosta problema.

* [Factory patern (Wikipedia)](https://en.wikipedia.org/wiki/Factory_pattern)

## Singleton

Pri razvoju web aplikacija, često ima smisla konceptualno i arhitekturalno omogućiti korištenje samo jedne
instance određene klase. Singleton patern omogućava upravo tako nešto.

{% highlight php %}
<?php
class Singleton
{
    /**
     * @var Singleton Referenca *Singleton* instance ove klase.
     */
    private static $instance;

    /**
     * Vraća *Singleton* instancu ove klase.
     *
     * @return Singleton The *Singleton* instance.
     */
    public static function getInstance()
    {
        if (null === static::$instance) {
            static::$instance = new static();
        }

        return static::$instance;
    }

    /**
     * Konstruktor je protected kako bi izvan klase bilo onemogućeno
     * kreiranje *Singleton* instance preko `new` operatora.
     */
    protected function __construct()
    {
    }

    /**
     * Sprječavanje kloniranja *Singleton* instance.
     *
     * @return void
     */
    private function __clone()
    {
    }

    /**
     * Sprječavanje deserijalizacije *Singleton* instance.
     *
     * @return void
     */
    private function __wakeup()
    {
    }
}

class SingletonChild extends Singleton
{
}

$obj = Singleton::getInstance();
var_dump($obj === Singleton::getInstance());             // bool(true)

$anotherObj = SingletonChild::getInstance();
var_dump($anotherObj === Singleton::getInstance());      // bool(false)

var_dump($anotherObj === SingletonChild::getInstance()); // bool(true)
{% endhighlight %}

Ovaj kôd implementira singleton patern koristeći [*statičku* promjenjivu](http://php.net/language.variables.scope#language.variables.scope.static)
i statički `getInstance()` metod. Obratite pažnju i na slijedeće:

* Konstruktor [`__construct()`](http://php.net/language.oop5.decon#object.construct) je deklariran kao protected
kako bi izvan klase bilo onemogućeno kreiranje *Singleton* instance preko `new` operatora.
* Magični metod [`__clone()`](http://php.net/language.oop5.cloning#object.clone) je deklariran kao private
kako bi bilo onemogućeno kloniranje instance preko [`clone`](http://php.net/language.oop5.cloning) operatora.
* Magični metod [`__wakeup()`](http://php.net/language.oop5.magic#object.wakeup) je deklariran kao private
kako bi bila onemogućena deserijalizacija instance preko globalne funkcije [`unserialize()`](http://php.net/function.unserialize).
* Nova instanca se kreira preko [late static binding](http://php.net/language.oop5.late-static-bindings) mehanizma
u statičkom metodu `getInstance()` putem ključne riječi `static`. Upravo ovo omogućava nasljeđivanje klase `Singleton` u primjeru.

Signleton patern je koristan u situacijama kada trebamo osigurati da imamo samo jednu instancu neke klase
u toku jednog kompletnog ciklusa u aplikaciji. Tipičan primjer su globalni objekti (kao što je neka Configuration klasa)
ili dijeljeni resursi (kao što je event queue).

Trebate biti oprezni pri korištenju Singleton paterna, jer sama njegova priroda uvodi globalno stanje
u vašu aplikaciju, čime se smanjuje njena testabilnost. U većini slučajeva, dependency injection princip može (i treba)
se koristiti umjesto Singleton klasa. Korištenjem dependency injection ne uvodimo nepotrebne direktne zavisnosti u
dizajn naše aplikacije, jer objekt koji će koristiti taj neki dijeljeni ili globalni resurs neće imati znanja o
tome o kojoj se točno klasi radi.

* [Singleton patern (Wikipedia)](https://en.wikipedia.org/wiki/Singleton_pattern)

## Strategy

Primjenom Strategy paterna enkapsulirate grupu određenih algoritama, pri čemu je klijentska klasa odgovorna za
instanciranje konkretnog algoritma, bez znanja o načinu na koji je on implementiran. Postoji nekoliko varijacija
ovog paterna, a najjednostavniji od njih će biti demonstriran u nastavku.

Prvi dio prikazuje grupu algoritama za ispis nekog niza podataka: jedan vrši nativnu, drugi radi JSON
serijalizaciju, a treći ga ostavlja netaknutim:

{% highlight php %}
<?php

interface OutputInterface
{
    public function load();
}

class SerializedArrayOutput implements OutputInterface
{
    public function load()
    {
        return serialize($arrayOfData);
    }
}

class JsonStringOutput implements OutputInterface
{
    public function load()
    {
        return json_encode($arrayOfData);
    }
}

class ArrayOutput implements OutputInterface
{
    public function load()
    {
        return $arrayOfData;
    }
}
{% endhighlight %}

Enkapsuliranjem ovih algoritama u zasebne klase, na elegantan i čist način stavljate do znanja drugim programerima
da lako mogu dodati novu output strategiju, bez utjecaja na klijentski kôd.

Primjetit ćete da svaka 'output' klasa implementira određeni OutputInterface. Ovaj interface ima za cilj ima da
definira jednostavan "ugovor" koji svaka nova implementacija mora ispoštovati. Također, implementiranjem jednog
zajedničkog interfacea, kao što ćete i vidjeti u nastavku, bit će omogućena primjena [Type Hinting-a](http://php.net/language.oop5.typehinting),
kako bi se osiguralo to da kôd koji koristi ovu funkcionalnost radi sa ispravnim tipovima klasa, u ovom slučaju 'OutputInterface' implementacijama.

Sljedeći primjer kôda demonstrira kako poziv klijentske klase može da koristiti neki od ovih algoritama
tako što će ga zahtjevati prilikom izvršavanja:

{% highlight php %}
<?php
class SomeClient
{
    private $output;

    public function setOutput(OutputInterface $outputType)
    {
        $this->output = $outputType;
    }

    public function loadOutput()
    {
        return $this->output->load();
    }
}
{% endhighlight %}

Ova klijentska klasa ima privatno svojstvo koje mora biti proslijeđeno prilikom instanciranja,
pri čemu to mora biti 'OutputInterface' implementacija. Nakon što je ovo svojstvo postavljeno,
poziv loadOutput() metoda će pozvati load() metod neke konkretne Output klase.

{% highlight php %}
<?php
$client = new SomeClient();

// Želite niz?
$client->setOutput(new ArrayOutput());
$data = $client->loadOutput();

// Želite JSON?
$client->setOutput(new JsonStringOutput());
$data = $client->loadOutput();

{% endhighlight %}

* [Strategy patern (Wikipedia)](http://en.wikipedia.org/wiki/Strategy_pattern)

## Front Controller

U slučaju Front Controller paterna postoji jedinstvena ulazna točka u vašoj aplikaciji (npr. index.php)
koja prihvaća sve zahtjeve. Taj dio kôda je odgovoran za učitavanje svih zavisnosti, procesiranje samog zahtjeva
i slanja odgovora browseru. Ovaj patern može biti vrlo koristan iz razloga što pospješuje modularan kôd
i osigurava centralno mjesto za ubacivanje nekog kôda koji se treba izvršavati na svaki zahtjev (kao na primjer
saniranje ulaznih podataka).

* [Front Controller patern (Wikipedia)](https://en.wikipedia.org/wiki/Front_Controller_pattern)

## Model-View-Controller

Model-View-Controller (MVC) patern i srodni paterni kao što su HMVC i MVVM vam omogućavaju da podijelite kôd
na logičke objekte pri čemu svaki ima posebnu namjenu. Model predstavlja sloj business logike i najčešće obavlja
manipulaciju podacima (dohvaćanje, čuvanje) koji se koriste u okviru aplikacije. Kontroleri (Controllers)
prihvaćaju zahtjev (request), uzimaju i obrađuju podatke od modela, a zatim učitavaju View-ove kako bi poslali odgovor (response).
View-ovi su templatei (markup, xml i slično) za prikaz podataka čiji se rezultirajući output šalje browseru
u okviru samog odgovora.

MVC je najčešće korišteni arhitekturalni patern u popularnim [PHP frameworcima](https://github.com/codeguy/php-the-right-way/wiki/Frameworks).

Naučite više o MVC-u i srodnim paternima:

* [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93View%E2%80%93Controller)
* [HMVC](https://en.wikipedia.org/wiki/Hierarchical_model%E2%80%93view%E2%80%93controller)
* [MVVM](https://en.wikipedia.org/wiki/Model_View_ViewModel)
