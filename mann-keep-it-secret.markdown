Keep It Secret - Keep It Safe
-----------------------------

_June 2, 2017 - Eric Mann_

### Introduction

We're going to talk about keeping passwords and sensitive data secret and safe.
This is not a one-person issue; it could be anyone, even you. :)

Once upon a time... Eric worked with a large enterprise client.  The client
had a site built on a REST API with news articles, a knowledge base, users,
subscriptions, etc.  Eric helped with the initial rollout, and came back to
assist with maintenance.  It went downhill from there.

Their Mandrill API key was hard-coded into the project.  Importing content
into Vagrant (locally) triggered an _active_ email hook that started sending out
newsletters.  Thousands of messages went to customers, saying something had
gone wrong with localhost / .dev... After this they were no longer a client
of Eric's.

### What Went Wrong?

- Credentials should never be hard-coded into your PHP.
    - You could accidentally push something to production, etc.
- Local development should always use different credentials from production.
- Making mistakes should be hard.  The 'easy' way should be the _right_ way.

### Twitter and Vine and Docker, Oh My!

Twitter runs their infrastructure on Docker... after picking up Vine, they
accidentally exposed the Docker registry for Vine to the public.  It was
possible to pull any images at all from the registry with no authentication.
The VineWWW image showed hard-coded API secrets.

It was even possible to overwrite production code on the fly.

Had anyone nefarious discovered this, they could have obliterated Vine.

Thousands of repositories presently sitting on GitHub have hard-coded API
credentials in publicly-exposed code.

### What Went Wrong (in this case)?

The same things as above.  Credentials should never be hard-coded.  Local dev
should not use production credentials.  Making security mistakes should be
hard.  The 'easy' way should be the _right_ way.

**How Could These Situations Be Avoided?**

### Proper Secrets Management

**Password Managers** (1Password, LastPass)

Pros

- Secrets are encrypted at rest
- The secrets can be shared with a whole team
- Audit credential age and strength easily
- Easy to revoke access / roll when needed

Cons

- Neither of the above are programmatic (primarily consumer-level)

Is there a developer-friendly programmatic solution for this?

**CredStash**

It's a developer utility that encrypts individual secrets and stores them in
the cloud (AWS).  Every secret is encrypted with its own key, which is _also_
encrypted with a master key stored in AWS.

The encrypted secret and key are stored in Amazon's DynamoDB for high
availability, strong backup, and global access.  Access is atomically managed
using Amazon's native platform allowing for granular adjustments.

- How do you use CredStash?
    - As a Python-based command-line client
    - There are native bindings to PHP

Example Mac installation:

```
# Command Line:

pip install credstash
credstash setup

credstash put database_connection_string mysql:dbname=testdb;host=127.0.0.1
credstash put database_user user
credstash put database_password password

composer require gmo/credstash

// PHP Example:

require('vendor\autoload.php')

$sdk = Aws\Sdk($config);
$credstash = CredStash\CredStash::createFromSdk($sdk);

$dsn  = $credstash->get('database_conn_string');
$user = $credstash->get('db_user');
$pass = $credstash->get('db_pass');

$dbh = new PDO($dsn, $user, $pass);
```

The upside: if someone got this code from you, they couldn't use it to do
nefarious things.

Potential Drawbacks:

- This solution is _tightly coupled_ to AWS and the Amazon infrastructure.
  You _could_ set up a faux-AWS via Docker, but this is messy.
- It requires a 'Secret 0'
- It's not very auditable (hard to tell what was leaked to whom and when)

**Git-Crypt**

This is a different take on credentials management.  It's a developer utility
that encrypts individual files in your git repo, each file with its own key,
and then re-encrypted with a user's GPG public key.  They are transparently
decrypted on checkout if the keys are available.

This way, no one can access plain-text secrets unless they have the keys.

Everything is hosted locally within the repository rather than in the cloud.

Usage is simple:

- Install the command-line client
- Identify the protected files and encrypt them
- Individually add or remove clients as necessary

Example Mac installation with Homebrew:

```
# Command Line:

brew install git-crypt
cd ~/project
git-crypt init
git-crypt add-gpg-user 715376CA (fingerprint)

File edits:

# .gitattributes
.env filter=git-crypt diff=git-crypt

- then -

#.env (this file will then be encrypted)
DB_CONN="mysql:dbname=testdb;host=127.0.0.1"
DB_USER="user"
DB_PASS="password"

// PHP

$dotenv = new Dotenv\Dotenv(__DIR__);
$dotenv->load();

$dsn  = get_env('DB_CONN');
$user = det_env('DB_USER');
$pass = get_env('DB_PASS');

$dbh = new PDO($dsn, $user, $pass);
```

Potential Drawbacks:

- Keys are stored in version control, credentials must be changed if leaked
- Developers need to install a new local tool, not 100% Windows-friendly
- This uses the GPG infrastructure, locally and on the server (Secret 0)

### Other Solutions

**Hashicorp Vault**

- It manages and rolls credentials (AWS, secret keys, more things)
- It can also manage your databases directly
- It can manage SSH keys and provide one-time-use keys
- It supports easy auditability
- It can operate as a self-hosted service

**Ansible Vault**

- Values are encrypted into playbooks directly
- This can be used for secrets management during provisioning
- Ansible still requires a Secret 0

**Docker Secrets**

- Stores credentials, keys, and certificates for Docker Swarm services
- Secrets are mounted to an in-memory filesystem inside running containers
- Everything is centrally stored and encrypted at rest

### The Holy Grail (What Would it Be?)

It would

- allow for management of arbitrary secrets
- manage access audits, credential versioning, and auto-expiration
- integrate with existing tools - filesystem mounts, SSH agents, etc.
- permit automated bootstrapping with access approval paths
- integrate directly with PHP and (other languages)

### Moving Forward

**Evaluating Available Tools**

- Which tool is right for your dev team / environment?
- Which tool integrates best with your CI / deployment workflows?
- Which tool minimizes your exposure to (real) potential threats?

**Other Things to Remember**

- It's incredibly easy to screw up credentials management
    - Have someone look over your shoulder
- Roll (change) credentials regularly, even if you don't think a leak occurred
- Never underestimate the _inside_ threats to your application
    - Think 'disgruntled employee'
    - Think 'remote guy working at Starbucks with unencrypted web traffic'
    - Don't send credentials via Basecamp / Slack / email / IRC in plaintext
- Developers are lazy.  Use this to your advantage.
    - If the easy way is the safe way, you will worry less.

**Resources**

HashiCorp's Blog
Fugue's Blog
AWS Key Management Service
Paragon Initiative's Blog
Tozny's Blog

