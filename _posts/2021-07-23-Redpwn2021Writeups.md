# redpwn2021 Writeups

## Web/Secure
This was a relatively simple challenge. The javscript in the page simply encodes input into base64 and then sends the query to the server. By manually sending unencoded text, an SQLI attack is possible, thus giving us the flag.

First, a quick look at the code executed when the form is submitted.
   ```
   <script>
    (async() => {
      await new Promise((resolve) => window.addEventListener('load', resolve));
      document.querySelector('form').addEventListener('submit', (e) => {
        e.preventDefault();
        const form = document.createElement('form');
        form.setAttribute('method', 'POST');
        form.setAttribute('action', '/login');

        const username = document.createElement('input');
        username.setAttribute('name', 'username');
        username.setAttribute('value',
          btoa(document.querySelector('#username').value)
        );

        const password = document.createElement('input');
        password.setAttribute('name', 'password');
        password.setAttribute('value',
          btoa(document.querySelector('#password').value)
        );

        form.appendChild(username);
        form.appendChild(password);

        form.setAttribute('style', 'display: none');

        document.body.appendChild(form);
        form.submit();
      });
    })();
  </script>
   ```
Make note the ```btoa``` function! It's used to encode a string as base64. This function is called on both the username and password. We can observe this by trying to log in with the username "admin" and the password "password" as seen below.
![](/images/Capture2.PNG)
Now, lets look at the server's code that's relevant to us:
```
app.post('/login', (req, res) => {
  if (!req.body.username || !req.body.password)
    return res.redirect('/?message=Username and password required!');

  const query = `SELECT id FROM users WHERE
          username = '${req.body.username}' AND
          password = '${req.body.password}';`;
  try {
    const id = db.prepare(query).get()?.id;

    if (id) return res.redirect(`/?message=${process.env.FLAG}`);
    else throw new Error('Incorrect login');
  } catch {
    return res.redirect(
      `/?message=Incorrect username or password. Query: ${query}`
    );
  }
});
```
Specifically, this bit:
```
const query = `SELECT id FROM users WHERE
          username = '${req.body.username}' AND
          password = '${req.body.password}';`;
```
This bit of code runs an SQL query like so:
    SELECT id FROM users WHERE '{our username input}' AND '{our password input}';

By using our knowledge of basic SQL, it's clear that inputting something like " 'or 1==1-- " into the password field would make the query return true.
Let's test that using Burpsuite to modify the request:
![](/images/Capture4.PNG)
![](/images/Capture5.PNG)
Success! 
