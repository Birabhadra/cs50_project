# SafePass – A password manager!
YouTube link: https://youtu.be/79Kk_gav9Vs
> [!IMPORTANT]
> Please, do not store your real passwords into this program!

> [!NOTE]
> I made the decision not to deploy this project for a straightforward reason: I'm not a cryptographer. My approach involved following library instructions and attempting to be sensible while developing this web application. The immense complexity in this field, coupled with the ever-evolving threats and attacks driven by advancements in technology and computational power, led me to the conclusion that I shouldn't solely depend on users to act reasonably by heeding my advice not to store their actual passwords in this program.  

## Introduction
This repository was submitted as the final project for CS50X 2025 online course offered by Harvard University. It’s a web application built around Flask’s framework and some software development/web programming languages covered by the course, such as Python, JavaScript, SQL, HTML and CSS. The application is designed for storing passwords.

The central component of the application is the **password vault**, where the user can **store**, **retrieve**, **upload** and **delete** their credentials from other sites or apps, which encompass domains, usernames, and passwords. To enable these interactions, the main application is **supported by additional files**, that contain **utilities** and **security** functions. 

Here's an overview of the interface:

![Application interface overview](static/Images/safepass_overview.png)

The project files follow the layout guidelines of [Flask](https://flask.palletsprojects.com/en/2.3.x/tutorial/layout/), but in a simplified manner. The root files are `app.py`, `helpers.py`, `security.py` and `schema.sql`, serving as the **backend** of the application. Regarding the **frontend**, it is divided into two folders: `static` and `templates`. These folders host user-interactive functionality scripts, a style sheet and svgs, and HTML content, respectively.

### Hashing and Encryption
When handling **passwords, storing them directly in the database is _never_ advisable**. Instead, it is essential to hash the password using a robust hashing function. This process renders the password nearly impossible to retrieve without the actual knowledge of the password itself. 

With that in mind, passwords provided by the user during the registration process for the application are hashed through the Argon2[^1] algorithm via a [CFFI library](https://argon2-cffi.readthedocs.io/en/stable/argon2.html).

Regarding user credentials such as domains, usernames and passwords for other sites or applications, an encryption method is applied to ensure their secure[^2] storage in the database. The encryption algorithm is provided by the [Fernet module]( https://cryptography.io/en/latest/fernet/) from the [cryptography library]( https://cryptography.io/en/latest/).

It involves a symmetric[^3] encryption that requires a passphrase and a random salt to derive a key. This key is used to encrypt data that can only be decrypted by the same key. The uniqueness of a key is directed related to the salt. Consequently, **it is _extremely_ important to _never_ reuse a salt for generating other keys**. This precaution is essential to maintain the security and integrity of the encryption process.

However, for this method to function properly, it requires the storage of the salt used during the key derivation process. This ensures that the same key gets generated by the server, thereby guaranteeing consistent access to the user’s data each time they log in.

Therefore, **to prevent the reuse of the salt**, and subsequently the same encryption key, **a new key is derived each time a user logs in**. The server accomplishes this by decrypting all of the user’s data with the old key during login, deriving a new one with a fresh randomly generated salt, and then encrypting it with the new key. This enhances the security of the process, but leaves the salt vulnerable in the case of a database leak.

To avoid this, the **salt is encrypted** with the `SERVER_KEY` by the same method.

The topics discussed above are illustrated below:
![Flowchart part one](static/Images/fluxograma_p1.PNG)

![Flowchart part two](static/Images/fluxograma_p2.PNG)

![Flowchart part three](static/Images/fluxograma_p3.PNG)

### Database schema
[Flask’s integration with SQLite3]( https://flask.palletsprojects.com/en/2.3.x/patterns/sqlite3/) was employed to serve as the [project’s database](/schema.sql). It’s a simple relational database featuring a table `users` with a one-to-many relationship with the table `passwords`.

![Illustration diagram for database schema](static/Images/database_diagram.png)

## Backend
### [`app.py`](/app.py)
This file is where most of the server logic resides. The first lines of code (1-25) start with the usual import declarations, some server configurations, the definition of the server Fernet key, which is a binary string used for encryption, and a function for closing the database connection when the application context[^4] is popped.

The next lines of code refer to the different routes of the application:
- [`register`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L123)
- [`login`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L199)
- [`index`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L34)
- [`import_file`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L418)
- [`change_password`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L288)
- [`change_master_password`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L337)
- [`delete_account`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L471)
- [`logout`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L514)

The route names are quite self-explanatory, and some of them share similar code patterns. Instead of providing detailed explanations for each route, I will describe the common patterns they follow while also highlighting any specific features or nuances.

#### `User input validation`
**User input and behavior should never be trusted**. This code pattern is present across all routes (except [`logout`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L514)) and is intended to safeguard the system against malicious user inputs, specifically SQL injection attacks and incorrect/incomplete filling of HTML forms. This is achieved by incorporating placeholders in SQL queries and implementing server-side validation.

The SQL queries are executed by a function called [`query_db`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L25) that allows the use of placeholders for queries. The implementation details can be found at [Flask’s website](https://flask.palletsprojects.com/en/2.3.x/patterns/sqlite3/#:~:text=def%20query_db(query%2C%20args%3D()%2C%20one%3DFalse)%3A).

Regarding the HTML forms, every route that is accessed via `POST` begins with a sequence of “if statements” that assess whether the forms have been submitted and whether the provided values are valid. One particular validation used at the [`register`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L123) route is done by a function called [`valid_password`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L60). A password and master password are only deemed valid by this function if they contain at least twelve characters, one uppercase letter, one lowercase letter, one number and one special character.

Another specific validation run by the program is done at the [`import_file`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L418) route. In this context, the system must verify whether the file selected by the user is in the right format (CSV files only). The function [`allowed_extension`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L50) manages this process, and further implementation details can be found at [Flask’s website]( https://flask.palletsprojects.com/en/2.3.x/patterns/fileuploads/#:~:text=def%20allowed_file(filename)%3A).

#### `User feedback`
While server-side validation is important to maintain system integrity, **providing users with feedback when they make a mistake or when an error occurs is equally crucial to ensure a satisfying user experience**.

User feedback is managed through the use of the native Flask functions `flash` and `get_flashed_messages`, as show below:
```
# Redirect user to home page
flash('Registered!', 'primary')
return redirect('/')
```

```
# Return to index
flash('Master password changed!', 'success')
return redirect('/')
```

```
# Ensure username was submitted
if not request.form.get('username'):
flash('Username is missing!', 'warning')
return redirect('/')
```

```
# Ensure password confirmation matches password
elif request.form.get('password') != request.form.get('confirmation'):
flash('Passwords does not match!', 'danger')
return render_template('register.html')
```
These code snippets illustrate how the flash function conveys information about the status of the route's procedures. The first argument represents the message to be displayed, while the second argument determines the message type, which, in turn, influences the color of the pop-up. 

Success messages appear in green, warnings in yellow, primary messages in blue, and danger messages in red.

Below is the HTML code section responsible for the actual display of the messages to the user. It utilizes [Bootstrap’s alert components](https://getbootstrap.com/docs/5.3/components/alerts/) via Jinja syntax.
```
{% if get_flashed_messages() %}
            <header>
                <div class="alert alert-{{ (get_flashed_messages(with_categories=True))[0][0] }} mb-0 text-center" role="alert">
                    {{ get_flashed_messages() | join(" ") }}
                </div>
            </header>
{% endif %}
```
For more information about flashed messages, check [here]( https://flask.palletsprojects.com/en/2.3.x/patterns/flashing/).

#### `DELETE and UPLOAD HTTP methods`
These methods are employed by the [`index`](https://github.com/abfarias/CS50-Final-Project/blob/main/app.py#L34) route, allowing users to both delete and edit their credentials from other sites or applications without the need of another route. This was achieved by sending information to the server via JSON format with the JavaScript method `fetch`[^5].

The information is accessed through the `request.get_json` method, encompassing the credential `id` and the user’s chosen action (delete or update).

Here's the corresponding JavaScript and Python code for `DELETE`:
```
// Create confirmation interface for deleting a password
    const rows = document.getElementsByName('tbody-row')
    const deleteButtons = document.getElementsByName('delete')
    
    for (let i = 0, length = deleteButtons.length; i < length; i++) {

        const row = rows[i]
        const deleteButton = deleteButtons[i]
        deleteButton.addEventListener('click', () => {

            // Remenber password row
            const id = row.getAttribute('id')

            // Send a DELETE http request if user's confirmation is true
            // Adapted from: https://www.youtube.com/watch?v=cuEtnrL9-H0
            if (confirm('Are you sure you want to delete this password?')) {
                fetch('/', {
                    method: 'DELETE',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify({ id: id })
                })
                .then(response => {
                    if (response.ok) {
                        window.location.reload()
                    } else {
                        console.log('Error: ' + response.statusText)
                    }
                })
                .catch(error => {
                    console.log('Error: ' + error)
                })
            }
        })
    }
```
```
 # User reached route via DELETE
    elif request.method == 'DELETE':
        delete_info = request.get_json()
        
        if delete_info is not None:
            query_db('DELETE FROM passwords WHERE id = ?', [delete_info['id']])
            flash('Password deleted!', 'danger')
            return jsonify({'success': True}), 200
        
        else:
            return redirect('/')
```

The same logic can be applied to `UPDATE`.


#### `Checking for rehash`
This is a specific section within the `login` route, where the hashes associated with a user's passwords are processed through a function to determine whether updating the hashes are necessary. This update becomes essential when there are changes to the hashing function's parameters.

This function is provided by the [Argon2 CFFI library](https://argon2-cffi.readthedocs.io/en/stable/argon2.html), and is incorporated as demonstrated below:

```
 # Query database for username
        rows = query_db('SELECT * FROM users WHERE username = ?', [request.form.get('username')])

...

# Ensure hash is up to date
        if check_rehash(rows[0]['hash']):
            new_hash = generate_hash(request.form.get('password'))
            query_db('UPDATE users SET hash = ? WHERE id = ?', [new_hash, session['user_id']])
        elif check_rehash(rows[0]['m_hash']):
            new_m_hash = generate_hash(request.form.get('master_password'))
            query_db('UPDATE users SET m_hash = ? WHERE id = ?', [new_m_hash, session['user_id']])
```

#### `Key derivation steps`
As mentioned earlier, key derivation takes place during the registration process and is repeated every time a user logs in. 

The code from `register`:
```
# Derive encryption from master password
        salt = os.urandom(16)
        key = derive_key(request.form.get('master_password', type=str), salt) 
        
        session['user_key'] = key

        # Store the username, hashed passwords and encrypted salt into the database
        username = request.form.get('username')
        hash = generate_hash(request.form.get('password'))
        m_hash = generate_hash(request.form.get('master_password'))
        encrypted_salt = SERVER_KEY.encrypt(salt)

        query_db('INSERT INTO users (username, hash, m_hash, salt) VALUES(?, ?, ?, ?)', [username, hash, m_hash, encrypted_salt])

        # Remember which user has register
        id = query_db('SELECT id FROM users WHERE username = ?', [username], True)

        session['user_id'] = id['id']
```
And for `login`:
```
# Derive same encryption key generated in /register from master password
        encrypted_salt = query_db('SELECT salt FROM users WHERE id = ?', [session['user_id']], True)
        salt = decrypt(SERVER_KEY, encrypted_salt['salt'])
        key = derive_key(request.form.get('master_password', type=str), salt)

# Decryption
…

# Generate new salt for new encryption key
        new_salt = os.urandom(16)
        new_key = derive_key(request.form.get('master_password', type=str), new_salt)

        # Remenber user key
        session['user_key'] = new_key

        # Save new salt into database
        new_encrypted_salt = SERVER_KEY.encrypt(new_salt)
        query_db('UPDATE users SET salt = ? WHERE id = ?', [new_encrypted_salt, session['user_id']])

# Encryption
…
```

#### `Encryption steps`
For the encryption to work, an instance of the `Fernet` class has to be created using the encryption key, as shown below:
```
from cryptography.fernet import Fernet
key = Fernet.generate_key()
f = Fernet(key) 
```
However, **storing this instance in a session for later use across other routes is not viable**. Instead, the generated key must be assigned to a variable and then stored in the session. This approach creates the need to instantiate the Fernet key within the current route whenever there's a need to perform data encryption or decryption.

Having said that, a round of the encryption looks like this:
```
# Encrypt data
user_key = Fernet(session['user_key'])

username = encrypt(user_key, request.form.get('username', type=str))
domain = encrypt(user_key, request.form.get('domain', type=str))
password = encrypt(user_key, request.form.get('password', type=str))

# Save it into the database
query_db('INSERT INTO passwords (user_id, username, domain, hash) VALUES (?, ?, ?, ?)', [session['user_id'], username, domain, password])
```
As for decryption:
```
# Get updated list of passwords
list = query_db('SELECT id, username, domain, hash FROM passwords WHERE user_id = ?', [session['user_id']])

# Decrypt list items
user_key = Fernet(session['user_key'])

for item in list:
    item['username'] = decrypt(user_key, item['username']).decode('utf-8')
    item['domain'] = decrypt(user_key, item['domain']).decode('utf-8')
    item['hash'] = decrypt(user_key, item['hash']).decode('utf-8')
```
These processes are performed quite often and can be found at many `routes`.

### [`helpers.py`](/helpers.py)
This file contains several useful functions that aid the main file objectives. 
Here is the list: 
- [`get_db`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L10): Establishes a connection with the SQLite3 database
- [`make_dicts`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L16): Dictionary factory, returns rows in the database as dictionaries
- [`query_db`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L25): Allows for easy and safe execution of commands in the database 
- [`login_required`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L36): Decorates the routes so they can only be accessed when the user is logged in
- [`allowed_extensions`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L50): Checks if the file extension is allowed
- [`valid_password`](https://github.com/abfarias/CS50-Final-Project/blob/main/helpers.py#L60): Defines a set of rules a password must follow to be considered valid

All functions above are provided by Flask[^6], with the exception of `valid_password`.

### [`security.py`](/security.py)
**This file contains all the security-related functions, abstracting them away from the main file for better organization**.
These functions are quite straightforward, and their main purposes have been explained further above. For more information, check out [`Argon2 CFFI Library`](https://argon2-cffi.readthedocs.io/en/stable/argon2.html) and [`cryptography`](https://cryptography.io/en/latest/).

Here’s the list:
- [`generate_hash`](https://github.com/abfarias/CS50-Final-Project/blob/main/security.py#L20)
- [`check_password`](https://github.com/abfarias/CS50-Final-Project/blob/main/security.py#L25)
- [`check_rehash`](https://github.com/abfarias/CS50-Final-Project/blob/main/security.py#L34)
- [`derive_key`](https://github.com/abfarias/CS50-Final-Project/blob/main/security.py#L34)
- [`encrypt`](https://github.com/abfarias/CS50-Final-Project/blob/main/security.py#L34)
- [`decrypt`](https://github.com/abfarias/CS50-Final-Project/blob/main/security.py#L67)

## Frontend
### `static` and `templates`
These folders contain files related to the application's design. The `static` folder contains images, .js and .css files, while the `templates` folder houses .html files.

### Layout, JavaScript and CSS
The layout was adapted from the course’s final problem set (Finance) while adding new features, such as password visibility togglers, modals, confirmation and editing interfaces, and dropdown menus.  

The idea behind the design was to created an interactive table, where the user could perform almost any operation within the same route. This was achieved with the use of JavaScript methods and Bootstrap components 

The CSS file was not heavily utilized, mainly because Bootstrap provided appealing default styles. Nevertheless, some HTML elements required specific positioning adjustments, prompting occasional use of the `style` tag.

## Room for improvement and Vulnerabilities
### `The issue with symmetric encryption`
As it currently stands, even with the precautions taken to avoid storing unencrypted salts in the database and the generation of a new key with each login, the encryption remains symmetric. This means that if an attacker gains access to the key, they would still be able to decrypt the data and obtain the passwords from the user.
A better, but more complex, solution for this problem would be to implement an asymmetric encryption method. In this setup, both the sender and the receiver have their own private and public keys, allowing for the encryption and decryption of data without the exposure of the private keys in the database[^7].

### `The lack of Two-Factor Authentication (2FA)`
Nowadays, it has become a standard practice for all online services to incorporate 2FA. However, this feature is currently absent in this application, leaving users more vulnerable to an attacker with access to their credentials.
So, an obvious next step, in the event of a further project development, it would be the implementation of an 2FA system.

### `Server key is exposed`
The `SERVER_KEY` exposure in the main file is a potential security concern. If this application were to be deployed, a different file architecture would be required. This would involve the use of an application factory pattern and assigning the `SEVER_KEY` to the `SECRET_KEY` [Flask’s convention]( https://flask.palletsprojects.com/en/2.3.x/tutorial/factory/) for enchanced security.

### `Better responsiveness for smaller screens`
The main aspect of the layout is the interactive table. However, on smaller screens the table doesn’t resize properly and instead overflows beyond the right side of the screen. I tried to ease this situation by including the `table-responsive` [Bootstrap class]( https://getbootstrap.com/docs/5.3/content/tables/#responsive-tables), that allows tables to be scrolled horizontally. Nonetheless, a general resize would look and feel a lot better. But this issue was noticed after the application logic had been already built around the table.

### `Spaghetti JavaScript code`
Here's a code snippet from [`index.js`](static/index.js):
```
row.removeChild(domainCell)
row.removeChild(usernameCell)
row.removeChild(passwordCell)
row.removeChild(utilityCell)

const newDomainCell = document.createElement('td')
const domainInput = document.createElement('input')
domainInput.setAttribute('autocomplete', 'off')
domainInput.setAttribute('autofocus', 'true')
domainInput.setAttribute('class', 'form-control mb-0 mx-0 w-auto')
domainInput.setAttribute('value', `${domain}`)
newDomainCell.appendChild(domainInput)

const newUsernameCell = document.createElement('td')
const usernameInput = document.createElement('input')
usernameInput.setAttribute('autocomplete', 'off')
usernameInput.setAttribute('autofocus', 'true')
usernameInput.setAttribute('class', 'form-control mb-0 mx-0 w-auto')
usernameInput.setAttribute('value', `${username}`)
newUsernameCell.appendChild(usernameInput)

const newPasswordCell = document.createElement('td')
const passwordInput = document.createElement('input')
passwordInput.setAttribute('autocomplete', 'off')
passwordInput.setAttribute('autofocus', 'true')
passwordInput.setAttribute('class', 'form-control mb-0 mx-0 w-auto')
passwordInput.setAttribute('value', `${password}`)
newPasswordCell.appendChild(passwordInput)
```
I think it goes without a saying that this is not the best design...

But as this was the quickest and easiest solution I could find at the time, and considering the scope of this project, that's fine!

## Sources
- [CS50X](https://cs50.harvard.edu/x/2023/) and [CS50W](https://cs50.harvard.edu/web/2020/) course materials
- [Flask docs](https://flask.palletsprojects.com/en/2.3.x/)
- [Jinja docs](https://jinja.palletsprojects.com/en/3.1.x/)
- [Cryptography lib](https://cryptography.io/en/latest/)
- [Argon2 CFFI lib](https://argon2-cffi.readthedocs.io/en/stable/argon2.html)
- [Python docs](https://docs.python.org/3/)
- [Bootstrap docs](https://getbootstrap.com/docs/5.3/getting-started/introduction/)
- These YouTube videos for some specific JavaScript functionalities
    - [Copy to Clipboard using HTML, CSS & JavaScript](https://www.youtube.com/watch?v=9-vBx7F0lns) by Codingflag
	- [Learn Fetch API in 6 Minutes](https://www.youtube.com/watch?v=cuEtnrL9-H0) by Web Dev Simplified


[^1]: The Argon2 algorithm is the winner of the [Password Hashing Competition (PHC)]( https://www.password-hashing.net/), conducted between 2012 and 2015.
[^2]: I’m no expert in cryptography, I only followed the instructions of the module to performs these encryptions. So, when I say “secure” you should take it with a grain of salt, and please do not store your real passwords into this program!
[^3]: Symmetric encryption is a cryptographic technique in which the same key is used for both the encryption and decryption of a message or data.
[^4]: Application context is defined [here](https://flask.palletsprojects.com/en/2.3.x/appcontext/#the-application-context).
[^5]: [This](https://www.youtube.com/watch?v=cuEtnrL9-H0) is a simple video explaining how to use the `fetch` method.
[^6]: [^6]:[`get_db`, `make_dicts` and `query_db`](https://flask.palletsprojects.com/en/2.3.x/patterns/sqlite3/), [`login_required`](https://flask.palletsprojects.com/en/2.3.x/patterns/viewdecorators/#login-required-decorator), [`allowed_extensions`](https://flask.palletsprojects.com/en/2.3.x/patterns/fileuploads/#uploading-files)
[^7]: The user encryption key is not directly stored in the database. Instead, the hashed password and the encrypted salt, used in the key derivation process, are stored. This makes it reasonably challenging to recover the actual key. However, it's worth noting that other attack vectors could potentially compromise a user's password, especially if the user stores it insecurely.
