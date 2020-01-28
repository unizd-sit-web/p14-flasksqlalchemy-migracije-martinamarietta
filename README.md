# Predavanje 14: FlaskSQLAlchemy + Migracije
Predavanje se nastavlja na prethodno (u kojem smo se upoznali s SQLite bazom podataka i SQLAlchemy ekstenzijom), a pokazat ćemo kako povezati Flask view funkcije i forme s bazom podataka uz pomoć [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/) ekstenzije. U bazu podataka ćmo spremati korisnike, pregledavati ih, mijenjati i brisati.

U drugom dijelu predavanja pokazat ćemo rad s migracijskim skriptama za rad s bazom podataka. Koristit ćemo [Flask-Migrate](https://flask-migrate.readthedocs.io/) ekstenziju.

Ukupno treba proći 6 zadatka za kompletiranje predavanja.

## Zadatak 1 - pokrenuti projekt i dodati CSS temu
* Klonirati projekt, aktivirati virualnu okolinu, instalirati potrebne ekstenzije, podesiti potrebne Flask varijable, te pokrenuti projekt.
* Pregledati strukturu i funkcionalnost projekta. Projekt koristi [Bootstrap 4](https://getbootstrap.com/) web framework.
* Dodati neku drugu temu. Sa stranica [Bootswatch](https://bootswatch.com/) skinuti neku temu, spremit je u static/css mapu, te ju referencirati u base.html datoteci u bloku "styles". Npr:
```
<link rel="stylesheet" href="{{ url_for('static', filename='css/bootstrap-darkly.min.css') }}">
```

## Zadatak 2 - flask-sqlalchemy
* Instalirati flask-sqlalchemy:
```
pip install flask-sqlalchemy
```
* U app.py datoteku dodati slijedeći kod, te ga proučite:
```
import os
from flask_sqlalchemy import SQLAlchemy
basedir = os.path.abspath(os.path.dirname(__file__))
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'data.sqlite')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

class Role(db.Model):
    __tablename__ = 'roles'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(64), unique=True)
    users = db.relationship('User', backref='role', lazy='dynamic')
    def __repr__(self):
        return '<Role %r>' % self.name

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True)
    role_id = db.Column(db.Integer, db.ForeignKey('roles.id'))
    def __repr__(self):
        return '<User %r>' % self.username

@app.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.name.data).first()
        if user is None:
            user = User(username=form.name.data)
            db.session.add(user)
            db.session.commit()
            session['known'] = False
        else:
            session['known'] = True
        session['name'] = form.name.data
        return redirect(url_for('index'))
    return render_template('index.html', form=form, name=session.get('name'),known=session.get('known', False))
```
* U index.html predložak promijenite u:
```
<div class="page-header">
    <h1>Pozdrav {% if name %}{{ name }}{% else %}stranče{% endif %}!</h1>
    {% if not known %}
    <p>Drago mi je da smo se upoznali!</p>
    {% else %}
    <p>Drago mi je da ste se vratili!</p>
    {% endif %}
</div>
```
* Prije ponovnog pokretanja aplikacije pokrenite slijedeći kod u Flask konzoli kako biste kreirali SQLite bazu podataka, te ju pregledajte u programu "DB Browser (SQLite)":
```
(venv) flask shell
>>> from app import db
>>> db.create_all()
```
* Pokrenite ponovo aplikaciju, te unesite jednog ili više korisnika. Pogledajte u bazi da li su korisnici upisani.
* Pokrenite aplikaciju u VS Code-u, postavite "breakpoint" na prvi red funkcije index(), te pratite u "Debug" prozoru korak po korak (F10) vrijednosti varijabli i izvođenje programa.

## Zadatak 3 - Pregled korisnika
* Dodajte dva nova predloška
    * "users.html" koji izlistava sve korisnike,
    * "user.html" koji ispisuje podatak o korisniku.
* Dodati u base.html link na popis korisnika:
```
<li><a class="nav-link" href="{{ url_for('users')}}">Korisnici</a></li>
```
* Dodati novi view s rutom:
```
@app.route('/users')
def users():
    users = User.query.all()
    return render_template('users.html', users=users)
```
* Dodati u users.html predložak:
```
{% extends "base.html" %}
{% block title %}Korisnici{% endblock %}
{% block page_content %}
<div class="page-header">
    <h1>Korisnici</h1>
</div>
<div>
    <ul>
        {% for user in users %}
            <li>{{ user.username }}</li>            
        {% endfor %}
    </ul>
</div>
{% endblock %}
```
* Pokrenite aplikaciju i pogledajte da li su svi korisnici ispisani.
* Promijenite liniju s ispisom korisnika u link:
```
<li><a href="{{ url_for('user', id=user.id )}}">{{ user.username }}</a></li>
```
* Dodati user view:
```
@app.route('/user/<id>')
def user(id):
    user = User.query.get (id)
    return render_template('user.html', user=user)
```
* Dodati u user.html predložak:
```
{% extends "base.html" %}
{% block title %}Korisnik | {{ user.username }}{% endblock %}
{% block page_content %}
<div class="page-header">
    <h1>Korisnik: <b>{{ user.username }}</b></h1>
</div>
{% endblock %}
```
* Pokrenite aplikaciju i provjerite da se na klik korisnika otvara stranica s podacima o korisniku.

## Zadatak 4 – dodati stranicu za grešku 404
* Promijeniti get u get_or_404() i prikazati grešku, tj. dodati error404.html predložak s tekstom i funkciju:
```
@app.errorhandler(404)
def page_not_found(e):
	return render_template('error404.html'), 404
```

## Zadatak 5 – brisanje
* Dodati na "user.html" predložak link (botun) za brisanje:
```
<a class="btn btn-danger" href="{{ url_for('userdelete', id=user.id ) }}" role="button">Briši</a>
```
* Dodati delete user view:
```
@app.route('/userdelete/<id>')
def userdelete(id):
    user = User.query.get_or_404(id)
    db.session.delete(user)
    db.session.commit()
    flash('Korisnik uspješno pobrisan.')
    return redirect(url_for('users'))
```

## Zadatak 6 - migracije
* Instalirati flask-migrate:
```
pip install flask-migrate
```
* Dodajemo u app.py:
```
from flask_migrate import Migrate
migrate = Migrate(app, db)
```
* Pokrenuti naredbu koja inicira migracije te stvara migrations folder sa svim potrebnim datotekama:
```
flask db init
```
* Promijenite naziv "data.sqlite" baze kako bismo stvorili novu (ili ju pobrišite)
* Kreiramo inicijalnu skriptu koja služi za stvaranje baze podataka:
```
flask db migrate -m "initial migration"
```
* Pregledajte stvorenu skriptu u "migrations/versions" mapi.
* Stvorimo novu bazu prema našem Python SQLAlchemy modelu pokreta:
```
flask db upgrade
```
* Pregledajte novostvorenu bazu
* Dodajte novo polje u User klasu. Npr. email. Ponovite naredbe:
```
flask db migrate -m "dodan email "
flask db upgrade
```
* Ponovo otvorite bazu i primijetit ćete novo polje. Također dodana je i nova migracijska skripta.

## Domaći rad
Napravite ažuriranje korisnika, brisanje s potvrdom, upravljanje rolama, dodavanje korisnika u rolu, dodajte novu tablicu i sl.
