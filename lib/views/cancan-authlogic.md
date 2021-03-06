#### {% title "Autoryzacja z Devise + CanCan" %}

# TODO na Devise: Autoryzacja Devise + CanCan

<blockquote>
  <p>{%= image_tag "/images/ryan-bates.png", :alt => "[Ryan Bates]" %}</p>
  <p class="center">Ryan Bates</p>
</blockquote>

**CanCan is a simple authorization plugin that offers a lot of flexibility.**
Zamierzamy to sprawdzić, dodając autoryzację do Fortunki z autentykacją.

* [Authorization with CanCan](http://railscasts.com/episodes/192-authorization-with-cancan)
  ([ASCIIcast](http://asciicasts.com/episodes/192-authorization-with-cancan))
* [CanCan](http://github.com/ryanb/cancan) –
  simple authorization for Rails.

Zaczynamy od sklonowania Fortunki z autentykacja:

    git clone git://sigma.ug.edu.pl/~wbzyl/rails3-authlogic

Teraz zmieniamy katalog:

    cd rails3-authlogic

i tak jak to opisano w pliku *README* tworzymy
„tracking branch” *cancan*:

    git checkout --track origin/cancan

Pora na migrację:

    rake db:migrate

Teraz jesteśmy w miejscu w którym zakończyliśmy
implementację autentykacji z Authlogic.
Ale tym razem jesteśmy na gałęzi **cancan**, a nie na **master**.

I18n: komunikaty definiowane przez CanCan powinny
być wypisywane w języku polskim.
Jak to zrobić jest opisane w dokumentacji kontrolera
[ControllerAdditions](http://rubydoc.info/github/ryanb/cancan/master/CanCan/ControllerAdditions)


## Autoryzacja na podstawie roli

Wykorzystamy gem *CanCan* do dodania trzech ról do Fortunki.
Dopuszczamy sytuację, że użytkownik może być przypisany do kilku ról.

* admin — **może** wszystko
* moderator — **może**  edytować i usuwać **każdą** fortunkę
* author — **może** dodawać, edytować i usuwać **swoje** fortunki

Zaczynamy od dodania powiązania między modelami *User*
i *Fortune*. Inaczej role *moderator* i *author* nie
będą miały sensu.

Ponieważ każdy użytkownik może wpisać wiele fortunek, a każda
fortunka została wpisana przez jednego użytkownika
więc między *User* a *Fortune* mamy powiązanie **jeden do wielu**:

    :::ruby
    class Fortune < ActiveRecord::Base
      attr_accessible :quotation, :user_id
      belongs_to :user

    class User < ActiveRecord::Base
      has_many :fortunes

Kończymy implementację powiązania między modelami dodając
pole *user_id:integer* do tabeli *fortunes*:

    rails g migration AddUserIDToFortune user_id:integer
    rake db:migrate

Dodatkowo będziemy korzystać z gemu *role_model*.
Dlaczego? Wyjaśni się za chwilę.

Dlatego do pliku *Gemfile* dopisujemy:

    :::ruby
    gem 'cancan'
    gem 'role_model'

i instalujemy nowe gemy wraz z zależnościami:

    bundle install

O „role based authorization” można poczytać na
[CanCan Wiki](http://github.com/ryanb/cancan/wiki/Role-Based-Authorization).


### Zmiany w tabeli *users*

Informację o rolach będziemy zapisywać w tabeli *users*.
W tym celu dodamy do tabeli nową kolumnę *roles_mask:integer*.

    rails g migration AddRolesMaskToUser roles_mask:integer
    rake db:migrate

Następnie w modelu *User* dopisujemy trochę kodu z README
gemu *role_model*.

    :::ruby
    class User < ActiveRecord::Base
      include RoleModel

      attr_accessible :roles

      # optionally set the integer attribute to store the roles in,
      # :roles_mask is the default
      roles_attribute :roles_mask

      # declare the valid roles -- do not change the order if you add more
      # roles later, always append them at the end!
      roles :admin, :moderator, :author

Przyjrzymy się rolom na konsoli przerabiając przykłady
z [README](http://github.com/martinrehfeld/role_model):

    rails console --sandbox

Zaczynamy od utworzenia nowego użytkownika:

    :::ruby
    u = User.new

role assignment:

    :::ruby
    u.roles = [:admin]  # ['admin'] works as well
    => [:admin]

adding roles:

    :::ruby
    u.roles << :moderator

querying roles: get all valid roles that have been declared:

    :::ruby
    User.valid_roles

retrieve all assigned roles:

    :::ruby
    u.roles # aliased to role_symbols for DeclarativeAuthorization

check for individual roles:

    :::ruby
    u.has_role? :author  # has_role? is also aliased to is?

check for multiple roles:

    :::ruby
    u.has_any_role? :author, :moderator   # has_any_role? is also aliased to is_any_of?
    u.has_all_roles? :author, :moderator  # has_all_roles? is also aliased to is?

see the internal bitmask representation (3 = 0b0011):

    :::ruby
    u.roles_mask

## Umieszczamy przykładowe dane w bazie

Ułatwimy sobie pracę z częściowo zaimplementowaną autoryzacją
umieszczając w bazie trochę danych.

Zaczynamy od zdefiniowania trzech użytkowników:

    :::ruby db/seed.rb
    User.create :username => 'wlodek', :email => 'matwb@ug.edu.pl',
      :password=> '1234', :password_confirmation => '1234',
      :roles => [:admin]
    User.create :username => 'renia', :email => 'renia@example.pl',
      :password=> '1234', :password_confirmation => '1234',
      :roles => [:moderator]
    User.create :username => 'bazylek', :email => 'bazylek@cats.com',
      :password=> '1234', :password_confirmation => '1234',
      :roles => [:author]

W tej wersji Fortunki skorzystamy z cytatów ze strony
[The Quotations Page](http://www.quotationspage.com/subjects/).
Włodkowi przypiszemy dwa, a Bazylkowi cztery cytaty z tej strony:

    :::ruby db/seed.rb
    Fortune.create :quotation => "Men willingly believe what they wish.",
      :user_id => 1
    Fortune.create :quotation => "I have often regretted my speech, never my silence.",
      :user_id => 1
    Fortune.create :quotation => "All science is either physics or stamp collecting.",
      :user_id => 3
    Fortune.create :quotation => "Nothing shocks me. I'm a scientist.",
      :user_id => 3
    Fortune.create :quotation => "Science is organized knowledge.",
      :user_id => 3
    Fortune.create :quotation => "Security is a kind of death.",
      :user_id => 3

Dane z pliku *db/seeds.rb* „przepiszemy” do bazy za pomocą gotowego
zadanie *rake*:

    rake db:seed  # load the seed data from db/seeds.rb

Na konsoli sprawdzamy, czy coś poszło nie tak:

    :::ruby
    User.all
    Fortune.all
    User.find(1).has_role? :admin

## Podrasowujemy widoki

Zaczynamy od widoku częściowego, gdzie dopisujemy:

    :::rhtml app/models/users/_form.html.erb
    <p>
      <%= f.label :roles %><br />
      <% for role in User.valid_roles %>
        <%= check_box_tag "user[roles][]", role, @user.roles.include?(role) %>
        <%= role.to_s.humanize %><br />
      <% end %>
    </p>

Przy okazji zmieniamy walidację:

    :::ruby
    class User < ActiveRecord::Base
      acts_as_authentic do |c|
        #c.validates_length_of_password_field_options= {:within => 2..4}
        #c.validates_length_of_password_confirmation_field_options= {:within => 2..4}
        c.ignore_blank_passwords= true
      end

Dlaczego? Czy poniższy cytat
z [dokumentacji](http://rubydoc.info/github/binarylogic/authlogic/master/Authlogic/ActsAsAuthentic/Password/Config#ignore_blank_passwords-instance_method)
coś wyjaśnia?

„Think about a profile page, where the user can edit all of
their information, including changing their password. If they do not
want to change their password they just leave the fields blank. This
will try to set the password to a blank value, in which case is
incorrect behavior. As such, Authlogic ignores this.”

Czy taka walidacja jest lepsza?

    :::ruby
    acts_as_authentic do |c|
       c.merge_validates_length_of_email_field_options(:allow_blank => true)
       c.merge_validates_format_of_email_field_options(:allow_blank => true)
    end


## Zmiany w layoucie

W główce każdej strony umieścimy login zalogowanego użytkownika.
Informację o zalogowanym użytkowniku znajdziemy w sesji.
Jeśli w widoku strony głównej dopiszemy:

    :::rhtml
    <%= debug(UserSession.find) %>

albo, lepiej:

    :::rhtml
    <%= debug(current_user) %>

to po zalogowaniu się do aplikacji i przejściu na stronę główną
powinniśmy zobaczyć co Authlogic zapisał w sesji:

    :::yaml
    --- !ruby/object:User
    attributes:
      created_at: 2010-09-16 20:21:00.365203
      last_request_at: &id001 2010-09-16 21:08:31.265948 Z
      crypted_password: 2e2b....a6ba
      perishable_token: DtGcInxA8Do0Z2vJcFNX
      updated_at: &id002 2010-09-16 21:08:31.266530 Z
      username: wlodek
      current_login_ip: 127.0.0.1
      id: 1
      current_login_at: 2010-09-16 21:03:53.627695
      password_salt: GCKhdI6rry414DqhhQLY
      roles_mask: 1
      login_count: 6
      persistence_token: ebd3....591d
      last_login_ip: 127.0.0.1
      last_login_at: 2010-09-16 20:53:01.460084
      email: matwb@ug.edu.pl
    attributes_cache:
      last_request_at: *id001
      updated_at: *id002
    changed_attributes: {}

    destroyed: false
    marked_for_destruction: false
    new_record: false
    new_record_before_save: false
    password_changed:
    previously_changed: !map:ActiveSupport::HashWithIndifferentAccess
      last_request_at:
      - 2010-09-16 21:05:40.304001 Z
      - *id001
      perishable_token:
      - xUS96rH5cPRMXvSCQZ6
      - DtGcInxA8Do0Z2vJcFNX
      updated_at:
      - 2010-09-16 21:05:40.304566 Z
      - *id002
    readonly: false
    skip_session_maintenance: false

Stąd widzimy, że

    :::ruby
    current_user.username

to login zalogowanego użytkownika.

Zatem dopisujemy w *layout/application.html.erb*
poniżej linka *"Logout"*:

    :::rhtml
    Welcome <em><%= current_user.username %></em>


## Zmiany widoków index i show

Wygenerowanny widok to tabelka. Nie wygląda to zbyt ładnie.
Zamiast tabelki uzyjemy kilku elementów *div*.

W widoku *show* przy każdym cytacie
dopiszemy login użytkownika który dodał cytat.

Oto oryginalny widok *index*:

    :::rhtml app/views/fortunes/index.html.erb
    <table>
      <tr>
        <th>Quotation</th>
      </tr>
      <% for fortune in @fortunes %>
        <tr>
          <td><%= fortune.quotation %></td>
          <td><%= link_to "Show", fortune %></td>
          <td><%= link_to "Edit", edit_fortune_path(fortune) %></td>
          <td><%= link_to "Destroy", fortune, :confirm => 'Are you sure?', :method => :delete %></td>
        </tr>
      <% end %>
    </table>
    <p><%= link_to "New Fortune", new_fortune_path %></p>

A to ten sam widok po „liftingu”:

    :::rhtml app/views/fortunes/index.html.erb
    <% for fortune in @fortunes %>
      <p><%= fortune.quotation %>
      <p><%= link_to "Show", fortune %>
         <% if logged_in? %>
           | <%= link_to "Edit", edit_fortune_path(fortune) %>
           | <%= link_to "Destroy", fortune, :confirm => 'Are you sure?',
                      :method => :delete %>
         <% end %>
      </p>
    <% end %>
    <p><%= link_to "New Fortune", new_fortune_path %></p>

Na razie ukrywamy linki do edycji i usuwania przed niezalogowanymi
użytkownikami. **TODO** Docelowo zastosujemy tutaj role.

A to są poprawki do widoku *show*:

    :::rhtml app/views/fortunes/show.html.erb
    <p>
      <strong>Quotation </strong> added by <em><%= @fortune.user.username %></em>
    </p>
    <p>
      <%= @fortune.quotation %>
    </p>


## Zmiany w edycji fortunek

Do widoku częściowego dodamy możliwość edycji
użytkownika (użyjemy listy rozwijanej), który dodał cytat.

    :::rhtml app/views/fortunes/_form.html.erb
    <% form_for @fortune do |f| %>
      <%= f.error_messages %>
      <p>
        <%= f.label :quotation %>:<br />
        <%= f.text_area :quotation, :cols => 71, :rows => 6 %>
      </p>
      <p>
        Zmień użytkownika: <%= f.collection_select :user_id, User.all, :id, :login %>
      </p>
      <p><%= f.submit "Submit" %></p>
    <% end %>

Po co to robimy?
Na razie każdy zalogowany użytkownik może dodawany przez siebie cytat
przypisać innemu użytkownikowi. Później ograniczymy możliwość edycji
użytkownika do admina i moderatora.

Tyle przygotowań i poprawek w aplikacji. Teraz zabieramy się za
implementację autoryzacji.


# Implementujemy autoryzację

<blockquote>
  {%= image_tag "/images/authorization.jpg", :alt => "[Autoryzacja]" %}
</blockquote>

Zgodnie z tym co jest napisane w dokumentacji CanCan,
autoryzację programujemy tworząc nową klasę
i włączając do niej moduł *CanCan::Ability*:

    :::ruby app/models/ability.rb
    class Ability
      include CanCan::Ability

      def initialize(user)
      end
    end

gdzie do metody *initialize* przkazywany jest obiekt *user*.
Następnie w metodzie **initialize** określimy to co może każdy użytkownik.

Autor gemu sugeruje, aby powyższy kod umieścić w folderze
*app/models*.

Autoryzację będziemy implementować małymi kroczkami.
Zaczniemy od umożliwienia każdemu użytkownikowi
wykonania akcji *index* oraz *show* dla rekordów
z tabeli *fortunes*:

    :::ruby app/models/ability.rb
    class Ability
      include CanCan::Ability

      def initialize(user)
        user ||= User.new  # guest user
        can :read, Fortune
      end
    end

Co to oznacza? CanCan definiuje kilka użytecznych aliasów,
m.in. alias `:read`:

    :::ruby
    alias_action :index, :show, :to => :read
    alias_action :new, :to => :create
    alias_action :edit, :to => :update

i skrótów:

    :::ruby
    can :manage, Fortune  # has permissions to do anything to fortunes
    can :manage, :all     # has permission to do anything to any model


Uaktywniamy autoryzację wykomentowując wiersz z *before_filter*
i wpisując w kodzie kontrolera *FortunesController*:

    :::ruby
    class FortunesController < ApplicationController
      #before_filter :require_user, :only => [:new, :edit, :create, :update, :destroy]
      load_and_authorize_resource

Ponieważ, CanCan jest kontrolerem napisanym w stylu RESTful, więc możemy
usunąć wszystkie linjki z kodem (tylko liczba pojedyncza):

    :::ruby
    @fortune = ...

ponieważ „resource is already loaded and authorized”.

Jest jeszcze jedna rzecz, o której zapomnieliśmy:

    :::ruby
    def create
      @fortune.user = current_user # przypisz cytat do zalogowanego użytkownika

Teraz możemy już sprawdzić jak to działa. Logujemy się do Fortunki i
próbujemy wyedytować jakiś cytat. Po kliknieciu w link *Edit*
powinnismy zobaczyć coś takiego:

    CanCan::AccessDenied in FortunesController#edit
      You are not authorized to access this page.

Oczywiście, taki wyjątek powinnismy jakoś obsłużyć. Zrobimy to tak.
Do pliku *application_controller.rb* dopiszemy:

    :::ruby
    rescue_from CanCan::AccessDenied do |exception|
      flash[:error] = exception.message
      redirect_to root_url
    end

Sprawdzamy jak to działa. Klikamy link *Edit*. Jest lepiej?

Następną rzeczą którą zrobimy będzie ukrycie linków *Edit*
i *Destroy* przed użytkownikami którzy nie mają uprawnień do edycji
i usuwania fortunek.

Przy okazji będziemy musieli zamienić/usunąć kod, który
dodaliśmy przy okazji autentykacji. Skorzystamy z metody
*can?* (*cannot?*):

Zaczynamy od zmian w widoku *fortunes/index.html.erb*:

    :::rhtml
    <% for fortune in @fortunes %>
      <p><%= fortune.quotation %>
      <p><%= link_to "Show", fortune %>
         <% if can? :update, fortune %>
           | <%= link_to "Edit", edit_fortune_path(fortune) %>
         <% end %>
         <% if can? :destroy, fortune %>
           | <%= link_to "Destroy", fortune, :confirm => 'Are you sure?', :method => :delete %>
         <% end %>
      </p>
    <% end %>

    <% if can? :create, fortune %>
      <p><%= link_to "New Fortune", new_fortune_path %></p>
    <% end %>

Następnie w pliku *show.html.erb* poprawiamy akapit:

    :::rhtml
    <p>
      <% if can? :update, @fortune %>
        <%= link_to "Edit", edit_fortune_path(@fortune) %>
      <% end %>
      <% if can? :destroy, @fortune %>
        | <%= link_to "Destroy", @fortune, :confirm => 'Are you sure?', :method => :delete %>
      <% end %>
    </p>

### Doprecyzowujemy *Abilities*

Teraz zabierzemy się za doprecyzowanie *abilities* korzystając
z ról, które dodaliśmy w trakcie wstępnych przygotowań.

Zaczynamy od admina.

    :::ruby
    class Ability
      include CanCan::Ability

      def initialize(user)
        user ||= User.new    # guest user
        if user.has_role? :admin
          can :manage, :all
        else
          can :read, Fortune
        end
      end
    end

Sprawdzamy, jak to działa logując się jako admin (*wlodek* jest adminem)
i jako zwykły użytkownik (*bazylek*).

Teraz zabieramy się za autora.
Autor może dodawać, edytować i usuwać swoje cytaty:

    :::ruby
    class Ability
      include CanCan::Ability

      def initialize(user)
        user ||= User.new

        if user.has_role? :admin
          can :manage, :all
        elsif user.has_role? :author
          can :read, Fortune
          can :create, Fortune
          can [:update, :destroy], Fortune, :user_id => user.id
        else
          can :read, Fortune
        end
      end
    end

Na koniec zostawiliśmy rolę *moderatora*, który może tylko edytować
i usuwać cytaty.

    :::ruby
    if user.has_role? :admin
      can :manage, :all
    elsif user.has_role? :author
      can :read, Fortune
      can :create, Fortune
      can [:update, :destroy], Fortune, :user_id => user.id
    elsif user.has_role? :moderator
      can :read, Fortune
      can [:update, :destroy], Fortune
    else
      can :read, Fortune
    end


## TODO

Powinniśmy jeszcze zadbać o to, aby *autor* nie mógł przypisać nowej
fortunki innemu użytkownikowi.

Niestety, nie wystarczy ukryć rozwijalnej listy „Zmień użytkownika”
przed autorami. Użytkownik może przechwycić wysyłany formularz, na
przykład za pomocą rozszerzenia *Web Developer* lub *Tamper Data*
przeglądarki Firefox, i zmienić wysyłane dane.

Wydaje się, że najprostszym rozwiązaniem będzie zignorowanie
przesyłanego id użytkownika w *params[:fortune]* w metodach *update*
i *create*. Zamiast tego ustawimy dane użytkownika na *current_user*:

    :::ruby
    def update
      if current_user.role? :author
        # to samo co params[:fortune][:user_id] = current_user.id
        @fortune.user = current_user
      end
      if @fortune.update_attributes(params[:fortune])

    def create
      @fortune = Fortune.new(params[:fortune])
      @fortune.user = current_user  # przebijamy ustawienia z przesłanego formularza
      ...

Zanim ukryjemy rozwijalną listę select przed autorami, jeszcze jedna
poprawka w kodzie kontrolera:

    :::ruby
    def new
      @fortune.user = current_user  # ustaw użytkownika na liście select
    end

Sprawdzamy, czy wszystko działa. Jesli tak, to ukrywamy listę
rozwijaną przed autorami (ale nie adminami i moderatorami). W widoku
*fortunes/_form.html.erb* podmieniamy akapit z „Zmień użytkownika…” na:

    :::rhtml
    <% if !current_user.role?(:author) %>
    <p>
      Zmień użytkownika: <%= f.collection_select :user_id, User.all, :id, :login %>
    </p>
    <% end %>


## TODO: Kosmetyka widoków

Dwie rzeczy należałoby zmienić w widoku *fortunes/index.html.erb*:

1. Autorzy powinni po zalogowaniu widzieć tylko swoje cytaty.
2. Admini i moderatorzy powinni po zalogowaniu widzieć kto dodał cytat.

Dlaczego?

Na stronie [Upgrading 1.1 – cancan](http://wiki.github.com/ryanb/cancan/upgrading-to-11)
w sekcji **Fetching Records** jest przykład pokazujący jak
zaprogramować punkt 1. powyżej. W *fortunes_controller.rb* w metodzie
*indeks* podmieniamy:

    :::ruby
    @fortunes = Fortunes.accessible_by(current_ability)

Ale powyższy kod zwraca to samo co *Fortunes.all* — czyli nie działa (bug?).

Zamiast tego metodę indeks zaprogramujemy tak:

    :::ruby
    def index
      @fortunes = current_user.fortunes
    end

co po uwzględnieniu adminów, moderatorów oraz niezalogowanych
użytkowników daje:

    :::ruby
    def index
      if current_user && current_user.role?(:author)
        @fortunes = current_user.fortunes
      else
        @fortunes = Fortune.all
      end
    end

Czy ten kod jest równoważny z rozwiązaniem korzystającym z *accessible_by*
ze strony gemu CanCan?


Pozostaje punkt 2. Można to zrobić tak:

    :::rhtml
    <% for fortune in @fortunes %>
      <p>
        <%=h fortune.quotation %>
        <% if logged_in? && (current_user.role?(:admin) || current_user.role?(:moderator)) %>
          (added by <em><%= fortune.user.login %></em>)
        <% end %>
      </p>
