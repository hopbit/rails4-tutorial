#### {% title "ActiveRecord na konsoli" %}

<blockquote>
  {%= image_tag "/images/orm.jpg", :alt => "[Orm]" %}
  <p>
    The object/relational mapping problem is <i>hard</i>. […] If you have
    an application problem that maps well to a NoSQL data model – such
    as aggregates or graphs – then you can avoid the nastiness of
    mapping completely.
  </p>
  <p class="author">— Martin Fowler, <a hhref="http://martinfowler.com/bliki/OrmHate.html">OrmHate</a></p>
</blockquote>

Lista przykładów:

1. Why Associations?
1. Badamy powiązanie wiele do wielu na konsoli
1. Powiązania polimorficzne
1. Efektywne pobieranie danych z kilku tabel – Eager Loading
1. Rodzime typy baz danych
1. Sytuacje wyścigu, czyli *race conditions*

Warto przeczytać:

* Simone Carletti,
  [Understanding Ruby and Rails: Delegate](http://www.simonecarletti.com/blog/2009/12/inside-ruby-on-rails-delegate/)

Na konsoli Rails najczęściej wykonujemy operacje CRUD na modelach.
Ale można sprawdzić też kilka innych rzeczy, na przykład routing:

    :::ruby konsola
    app.fortunes_path
    app.get app.fortune_path(4)
    app.response.body
    app.response.headers["content-type"]

albo tak:

    :::ruby
    include Rails.application.routes.url_helpers
    fortune_path(2)
    fortunes_url(:host => "localhost:3000")

albo tak:

    :::ruby konsola
    Rails.application.routes.url_helpers.fortunes_path
    Rails.application.routes.url_helpers.fortunes_url(:host => "localhost:3000")

Metody pomocnicze:

    :::ruby
    helper.number_to_currency(9.99)

Więcej na temat sztuczek konsolowych:

* [Three quick Rails console tips](http://37signals.com/svn/posts/3176-three-quick-rails-console-tips)

Zaczniemy od sprawdzenia jakie mamy zainstalowane w systemie
Rubies i zestawy gemów:

    :::bash
    rvm list
    rvm list gemsets

Następnie, na potrzeby przykładów z tego wykładu, utworzymy zestaw
gemów o nazwie *active_record*:

    :::bash
    rvm use --create ruby-1.9.3-p194@active_record
    gem update
    gem install rails
    rvm info
    rvm current

Co daje takie podejście?

Dokumentacja:

* [ActiveRecord::Associations::ClassMethods](http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html)
* [ActiveRecord::Base](http://api.rubyonrails.org/classes/ActiveRecord/Base.html)


## Why Associations?

Przykład z rozdziału 1
[A Guide to Active Record Associations](http://guides.rubyonrails.org/association_basics.html).
Zobacz też [ActiveRecord::ConnectionAdapters::TableDefinition](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/TableDefinition.html).

Generujemy przykładową aplikację:

    :::bash
    rvm use ruby-1.9.3-p194@active_record
    rails new why_associations --skip-bundle --skip-test-unit
    cd why_associations
    bundle install --path=.bundle/gems

Przy okazji ułatwimy sobie śledzenie kolejności
migracji dopisując do pliku *config/application.rb*:

    :::ruby
    config.active_record.timestamped_migrations = false

Skorzystamy z generatora do wygenerowania kodu dla dwóch modeli:

* *Customer* (klient) – atrybuty: *name*, …
* *Order* (zamówienie) – atrybuty: *order_date*, *order_number* …

Każdy klient może mieć wiele zamówień. Każde zamówienie
należy do jednego kilenta. Dlatego między kilentami i zamówieniami
mamy powiązanie jeden do wielu. Oznacza to, że powinniśmy dodać
jeszcze jeden atrybut (*foreign key*) do zamówień:

* *Order* (zamówienie) – atrybuty: **customer_id**, order_date, …

Generujemy *boilerplate code*:

    :::bash
    rails generate model Customer name:string
    rails generate model Order order_date:datetime order_number:string customer:references

Migrujemy i przechodzimy na konsolę:

    :::rails
    rake db:migrate
    rails console

gdzie dodajemy klienta i dwa jego zamówienia:

    :::ruby
    Customer.create :name => 'wlodek'
    @customer = Customer.first  # jakiś klient
    Order.create :order_date => Time.now, :order_number => '20111003/1', :customer_id => @customer.id
    Order.create order_date: Time.now, order_number: '20111003/2', customer_id: @customer.id
    Order.all

A tak usuwamy z bazy klienta i wszystkie jego zamówienia:

    :::ruby
    @customer = Customer.first
    @orders = Order.where customer_id: @customer.id
    @orders.each { |order| order.destroy }
    @customer.destroy


### Dodajemy powiązania między modelami

Po dopisaniu kod ustalającego powiązania między tymi modelami:

    :::ruby
    class Customer < ActiveRecord::Base
      has_many :orders, :dependent => :destroy
    end

    class Order < ActiveRecord::Base
      belongs_to :customer
    end

tworzenie nowych zamówień dla danego klienta jest łatwiejsze:

    :::ruby
    Customer.create(name: 'rysiek')
    @customer = Customer.where(name: 'rysiek').first
    @order = @customer.orders.create order_date: Time.now, order_number: '20111003/3'
    @order = @customer.orders.create order_date: Time.now, order_number: '20111003/4'

Usunięcie kilenta wraz z wszystkimi jego zamówieniami jest też proste:

    :::ruby
    @customer = Customer.first  # jakiś klient
    @customer.destroy

Tak jest prościej.


## Badamy powiązanie wiele do wielu na konsoli

Między zasobami – *assets* i opisującymi je cechami – *tags*.

Przykład pokazujący o co nam chodzi:

* *Asset*: Cypress. A photo of a tree. (dwa atrybuty)
* *Tag*: tree, organic

Między zasobami i opisującymi je cechami mamy powiązanie wiele do
wielu.

Dodatkowo, każdy zasób przypisujemy do jednego z kilku rodzajów zasobów –
*AssetType*.  Przykład:

* *Asset*: Cypress. A photo of a tree.
* *AssetType*: Photo

Między zasobem a rodzajem zasobu mamy powiązanie wiele do jednego.

Tak jak poprzednio skorzystamy z generatora do wygenerowania
**boilerplate code**:

    :::bash
    rails g model AssetType name
    rails g model Asset name description:text asset_type:references
    rails g model Tag name

Dla powiązania wiele do wielu między *Asset* i *Tag*, zgodnie
z konwencją Rails, powinniśmy dodać tabelę o nazwie *assets_tags*
(nazwy tabel w kolejności alfabetycznej):

    :::bash
    # rails g model AssetsTags asset:references:index tag:references:index
    #   ale indeksy są tworzone automatycznie dla atrybutów *references*
    #   dlatego wystarczy
    rails g model AssetsTags asset:references tag:references

W migracji dopisujemy *id: false* (dlaczego? konwencja Rails)
i usuwamy zbędne *timestamps* (też wymagane przez Rails?):

    :::ruby
    class CreateAssetsTags < ActiveRecord::Migration
      def change
        create_table :assets_tags, id: false do |t|
          t.references :asset
          t.references :tag
        end
        add_index :assets_tags, :asset_id
        add_index :assets_tags, :tag_id
      end
    end

Dopiero teraz migrujemy i usuwamy niepotrzebny model
(w dowolnej kolejności):

    rm app/models/assets_tags.rb
    rake db:migrate

Na koniec, dodajemy powiązania do tych modeli:

    :::ruby
    class Asset < ActiveRecord::Base
      has_and_belongs_to_many :tags
      belongs_to :asset_type
    end

    class Tag < ActiveRecord::Base
      has_and_belongs_to_many :assets
    end

    class AssetType < ActiveRecord::Base
      attr_accessible :description, :name, :asset_type_id #<= dopisujemy
      has_many :assets
    end

Skorzystamy z zadania *rake* o nazwie *db:seed*
do umieszczenia danych w tabelach:

    :::ruby db/seeds.rb
    AssetType.create name: 'Photo'
    AssetType.create name: 'Painting'
    AssetType.create name: 'Print'
    AssetType.create name: 'Drawing'
    AssetType.create name: 'Movie'
    AssetType.create name: 'CD'

    Asset.create name: 'Cypress', description: 'A photo of a tree.',     asset_type_id: 1
    Asset.create name: 'Blunder', description: 'An action file.',        asset_type_id: 5
    Asset.create name: 'Snap',    description: 'A recording of a fire.', asset_type_id: 6

    Tag.create name: 'hot'
    Tag.create name: 'red'
    Tag.create name: 'boring'
    Tag.create name: 'tree'
    Tag.create name: 'organic'

Ale wcześniej usuniemy bazę:

    rake db:drop
    rake db:schema:load

Powiązania dodamy na konsoli Rails:

    :::ruby
    a = Asset.find 1
    t = Tag.find [4, 5]
    a.tags << t
    Asset.find(2).tags << Tag.find(3)
    Asset.find(3).tags << Tag.find([1, 2])

Chcemy zbadać powiązania między powyżej wygenerowanymi modelami.
Zabadamy powiązania z konsoli Rails.

Konsola Rails:

    :::ruby
    a = Asset.find(3)
    y a
    a.name
    a.description
    a.asset_type.name
    a.tags
    a.tags.each { |t| puts t.name } ; nil
    y a.asset_type
    y a.tags
    a.tags.each { |t| puts t.name } ; nil
    aa = Asset.first
    puts aa.to_yaml

Przyjrzeć się uważnie co jest wypisywane na terminalu.


### Jeszcze jeden przykład: Author & Article

Między autorami i artykułami mamy powiązanie wiele do wielu.
Dlaczego?

* Utworzyć model *Author* za pomocą generatora.
* Jak wyżej, ale utworzyć model *Article*.
* Na koniec, utworzyć model *Bibinfo* – informacja bibliograficzna.

Dodać do modeli następujące powiązania:

    :::ruby
    class Author < ActiveRecord::Base
      has_many :bibinfos
      has_many :articles, through: :bibinfos
    end

    # relacja wiele-do-wielu via Bibinfo model
    class Bibinfo < ActiveRecord::Base
      belongs_to :author
      belongs_to :article
    end

    class Article < ActiveRecord::Base
      has_many :bibinfos
      has_many :authors, through: :bibinfos
    end

Przejść na konsolę. Zapisać w tabelkach następujące artykuły (książki):

* David Flanagan, Yukihiro Matsumoto. **The Ruby Programming Language**
  (ISBN-13: 978-0-59651-617-8, Publication Date: February 1, 2008, Edition: First Edition, Publisher: O'Reilly)
* Dave Thomas, Chad Fowler, Andy Hunt. **Programming Ruby 1.9**
  (Publication Date: April 15, 2009, ISBN: 978-1-93435-608-1, Edition: 3rd Edition)
* Sam Ruby, Dave Thomas, David Heinemeier Hansson. **Agile Web Development with Rails**
  (Published: 2011-03-31, ISBN: 978-1-93435-654-8, Pages: 448, Edition: 4th Edition)
* David Flanagan. **jQuery Pocket Reference**
  (Publication Date: December 2010, Pages: 160, Print ISBN: 978-1-4493-9722-7, Ebook ISBN: 978-1-4493-9732-6)

*Wskazówki:*
(i) [The has_many :through Association](http://guides.rubyonrails.org/association_basics.html#the-has_many-through-association),
(ii) {%= link_to "jak wygenerować modele?", "/database_seed/gen-models-why_associations.sh" %},
(iii) {%= link_to "seeds.rb", "/database_seed/seeds-why_associations.rb" %} (rozwiązanie).

Wypróbować na konsoli poniższe powiązania:

    :::ruby
    author.articles
    article.authors
    author.bibinfos
    article.bibinfos
    bibinfo.author
    bibinfo.article


## Powiązania polimorficzne

Typowy przykład. Trzy modele:

* *Person* (osoba)
* *Company* (firma)
* *PhoneNumber* (numer telefonu)

Osoba może mieć kilka numerów telefonów. Firma też.

Niestety w Railsach, nie możemy w kodzie modelu *PhoneNumber* wpisać
**kilkukrotnie** *belongs_to*

    :::ruby
    class PhoneNumber < ActiveRecord::Base
      belongs_to :person
      belongs_to :company

Ale możemy to ograniczenie obejść powielając model:

* *PersonPhoneNumber*
* *CompanyPhoneNumber*

Takie rozwiązanie prowadzi do powielania kodu,
co jest kardynalnym błędem. Do takich sytuacji
w Rails stworzono tzw. *polymorphic associations*.

Zaczynamy od wygenerowania trzech modeli:

    rails g model Person name:string
    rails g model Company name:string
    rails g model PhoneNumber number:string callable:references

Dopisujemy zależności do modeli:

    :::ruby
    class Person < ActiveRecord::Base
      has_many :phone_numbers, as: :callable, dependent: :destroy
    end
    class Company < ActiveRecord::Base
      has_many :phone_numbers, as: :callable, dependent: :destroy
    end

    class PhoneNumber < ActiveRecord::Base
      belongs_to :callable, polymorphic: true  #<= dopisujemy polymorphic…
    end

Poprawiamy migrację wygenerowaną dla *phone_numbers*:

    :::ruby
    class CreatePhoneNumbers < ActiveRecord::Migration
      def change
        create_table :phone_numbers do |t|
          t.string :number
          t.references :callable, polymorphic: true
          t.timestamps
        end
      end
    end

*Uwaga:* wiersz kodu z *polymorphic: true* powyżej zastępuje:

    :::ruby
    t.integer :callable_id
    t.string  :callable_type

Migrujemy:

    rake db:migrate

Teraz już możemy przećwiczyć polimorfizm na konsoli Rails:

    rails console

gdzie wykonujemy następujący kod:

    :::ruby
    p = Person.create :name => 'Wlodek'
    c = Company.create :name => 'Instytut Informatyki'
    p.phone_numbers << PhoneNumber.create(number: '58-000-0000')
    c.phone_numbers << PhoneNumber.create(number: '58-001-0001')
    PhoneNumber.all
      +----+---------+-------------+---------------
      | id | number  | callable_id | callable_type
      +----+---------+-------------+---------------
      | 1  | 58-..   | 1           | Person
      +----+---------+-------------+---------------
      | 2  | 58-..   | 1           | Company
      +----+---------+-------------+---------------
    p.phone_numbers
      +----+---------+-------------+---------------
      | id | number  | callable_id | callable_type
      +----+---------+-------------+---------------
      | 1  | 58-..   | 1           | Person
      +----+---------+-------------+---------------

Jak działa *p.phone_numbers*?

Podobny przykład znajdziemy w przewodniku
[A Guide to Active Record Associations](http://guides.rubyonrails.org/association_basics.html#polymorphic-associations).

Inne przykłady:

* [acts-as-taggable-on](https://github.com/mbleigh/acts-as-taggable-on)
* [acts_as_commentable](https://github.com/jackdempsey/acts_as_commentable) (tylko ActiveRecord)

Co w nich jest ciekawego?


## Efektywne pobieranie danych – Eager Loading

„Eager loading is the mechanism for loading the associated records of
the objects returned by *Model.find* using as few queries as possible.”

Powyższy cytat zaczyna sekcję
[Eager Loading Associations](http://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations)
przewodnika „Active Record Query Interface”.

W przewodniku przedstawiono prosty, składający się z dwóch modeli
*Client* i *Address*, przykład, a poniżej — nieco bardziej złożony
przykład składający się z trzech modeli: *Fotograf*, *Galeria*
i *Zdjęcie*.

Zależności między modelami są takie:

* *Fotograf* ma wiele *Galerii*
* *Galeria* ma wiele *Zdjęć*

Dow wygenerowania kodu skorzystamy z generatora *model*:

    rails g model Photographer name
    rails g model Gallery photographer:references name
    rails g model Photo gallery:references name file_path

Do wygenerowanego kodu dopisujemy powiązania:

    :::ruby
    class Photographer < ActiveRecord::Base
      has_many :galleries
    end

    class Gallery < ActiveRecord::Base
      attr_accessible :name, :photographer_id

      has_many :photos
      belongs_to :photographer
    end

    class Photo < ActiveRecord::Base
      attr_accessible :file_path, :name, :gallery_id

      belongs_to :gallery
    end

Na koniec, migrujemy:

    :::bash
    rake db:migrate

Dopisujemy kod do pliku *db/seeds.rb* (albo wykonujemy go bezpośrednio
na konsoli Rails):

    :::ruby db/seeds.rb
    Photographer.create name: 'Jan Saudek'
    Photographer.create name: 'Stefan Rohner'

    Gallery.create name: 'Nordic Light', photographer_id: 1
    Gallery.create name: 'Daily Life', photographer_id: 2
    Gallery.create name: 'India', photographer_id: 2

    Photo.create name: 'Shadows', file_path: 'photos/img_1154.jpg', gallery_id: 1
    Photo.create name: 'Ice Formation', file_path: 'photos/img_6836.jpg', gallery_id: 1
    Photo.create name: 'Unknown', file_path: 'photos/img_8419.jpg', gallery_id: 2
    Photo.create name: 'Uptown', file_path: 'photos/img_1243.jpg', gallery_id: 2
    Photo.create name: 'India Sunset', file_path: 'photos/img_2349.jpg', gallery_id: 2
    Photo.create name: 'Summer', file_path: 'photos/img_7744.jpg', gallery_id: 3
    Photo.create name: 'Two cats', file_path: 'photos/img_1440.jpg', gallery_id: 3
    Photo.create name: 'Dogs', file_path: 'photos/img_1184.jpg', gallery_id: 3

Na konsoli wykonujemy:

    :::ruby
    galleries = Gallery.includes(:photographer, :photos).all

Wykonanie tego polecenia skutkuje trzykrotnym odpytaniem bazy:

    Gallery Load (0.2ms)  SELECT "galleries".* FROM "galleries"
    Photographer Load (0.1ms)  SELECT "photographers".* FROM "photographers"
      WHERE "photographers"."id" IN (1, 2)
    Photo Load (0.2ms)  SELECT "photos".* FROM "photos"
      WHERE "photos"."gallery_id" IN (1, 2, 3)

Teraz polecenia:

    :::ruby
    galleries.each do |gallery|
      puts gallery.photographer.name, gallery.photos[0].name
    end

    galleries[0].photographer
    galleries[0].photos

nie odpytują bazy, ponieważ dane zostały wcześniej wczytane (*eager loading*).


## Rodzime typy baz danych

I jak są tłumaczone z typów danych języka Ruby.
Warto też przejrzeć przykłady
w [dokumentacji metody *column*](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/TableDefinition.html)

PostgreSQL:

    :::ruby activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb
    ADAPTER_NAME = 'PostgreSQL'

    NATIVE_DATABASE_TYPES = {
      primary_key: "serial primary key",
      string:      { name: "character varying", limit: 255 },
      text:        { name: "text" },
      integer:     { name: "integer" },
      float:       { name: "float" },
      decimal:     { name: "decimal" },
      datetime:    { name: "timestamp" },
      timestamp:   { name: "timestamp" },
      time:        { name: "time" },
      date:        { name: "date" },
      binary:      { name: "bytea" },
      boolean:     { name: "boolean" },
      xml:         { name: "xml" },
      tsvector:    { name: "tsvector" },
      hstore:      { name: "hstore" },
      inet:        { name: "inet" },
      cidr:        { name: "cidr" },
      macaddr:     { name: "macaddr" },
      uuid:        { name: "uuid" },
      json:        { name: "json" }
    }

SQLite:

    :::ruby activerecord/lib/active_record/connection_adapters/sqlite_adapter.rb
    def native_database_types
      {
        :primary_key => default_primary_key_type,
        :string      => { :name => "varchar", :limit => 255 },
        :text        => { :name => "text" },
        :integer     => { :name => "integer" },
        :float       => { :name => "float" },
        :decimal     => { :name => "decimal" },
        :datetime    => { :name => "datetime" },
        :timestamp   => { :name => "datetime" },
        :time        => { :name => "time" },
        :date        => { :name => "date" },
        :binary      => { :name => "blob" },
        :boolean     => { :name => "boolean" }
      }
    end


## Race Conditions

czyli tzw. **sytuacje wyścigu**.

Przykład: tworzymy dwa modele: *Inventory* (magazyn) i *Cart* (koszyk):

    :::bash
    rails g model Inventory name on_hand:integer
    rails g model Cart name quantity:integer

Oto wygenerowane i nieco poprawione migracje:

    :::ruby
    class CreateInventories < ActiveRecord::Migration
      def change
        create_table :inventories do |t|
          t.string :name
          t.integer :on_hand  # na stanie
        end
      end
    end

    class CreateCarts < ActiveRecord::Migration
      def change
        create_table :carts do |t|
          t.string :name
          t.integer :quantity, default: 0  # dodane ręcznie
        end
      end
    end

Do modelu *Inventory* dodajemy walidację:

    :::ruby inventory.rb
    class Inventory < ActiveRecord::Base
      validate :on_hand_could_not_be_negative

      def on_hand_could_not_be_negative
        errors.add(:on_hand, "can't be negative") if on_hand < 0
      end
    end

na koniec migrujemy:

    :::bash
    rake db:migrate

Przechodzimy na konsolę Ruby:

    :::ruby
    Inventory.create name: 'Laptop Eee PC 1000', on_hand: 10

Przykład pokazujący *race condition*
Dwóch klientów jednocześnie kupuje po 8 laptopów Eee PC 1000:

    :::ruby
    c1 = Cart.create name: 'Laptop Eee PC 1000', quantity: 8
    c2 = Cart.create name: 'Laptop Eee PC 1000', quantity: 8

Oczywiście, po tym jak pierwszy kilent dodał 8 laptopów do swojego
koszyka, zabraknie 6 laptopów dla drugiego klienta.
Dlatego, drugi klient powinien poczekać aż zostanie uaktualniony
stan magazynu.

Możemy to zaprogramować korzystając z transakcji i wyjątków:

    :::ruby
    laptop = 'Laptop Eee PC 1000'
    quantity = 8
    c1 = Cart.create name: laptop, quantity: quantity
    begin
      Cart.transaction do
        item = Inventory.find_by_name laptop
        item.on_hand -= quantity
        item.save!
      end
    rescue
      flash[:error] = "Sorka, laptopy właśnie się skończyły!"
    end

Podobnie postępujemy z drugim klientem. Oczywiście, powyższy
kod zamieniamy na metodę, np. o nazwie *add_item*,
którą dodajemy do *CartController*.
