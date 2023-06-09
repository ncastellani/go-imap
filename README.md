# go-imap

[![godocs.io](https://godocs.io/github.com/ncastellani/imapServer?status.svg)](https://godocs.io/github.com/ncastellani/imapServer)
[![builds.sr.ht status](https://builds.sr.ht/~emersion/go-imap/commits/master.svg)](https://builds.sr.ht/~emersion/go-imap/commits/master?)

An [IMAP4rev1](https://tools.ietf.org/html/rfc3501) library written in Go. It
can be used to build a client and/or a server.

> **Note**
> This is the README for go-imap v1. go-imap v2 is in development, see the
> [v2 branch](https://github.com/ncastellani/imapServer/tree/v2) for more details.

## Usage

### Client [![godocs.io](https://godocs.io/github.com/ncastellani/imapServer/client?status.svg)](https://godocs.io/github.com/ncastellani/imapServer/client)

```go
package main

import (
	"log"

	"github.com/ncastellani/imapServer/client"
	"github.com/ncastellani/imapServer"
)

func main() {
	log.Println("Connecting to server...")

	// Connect to server
	c, err := client.DialTLS("mail.example.org:993", nil)
	if err != nil {
		log.Fatal(err)
	}
	log.Println("Connected")

	// Don't forget to logout
	defer c.Logout()

	// Login
	if err := c.Login("username", "password"); err != nil {
		log.Fatal(err)
	}
	log.Println("Logged in")

	// List mailboxes
	mailboxes := make(chan *imap.MailboxInfo, 10)
	done := make(chan error, 1)
	go func () {
		done <- c.List("", "*", mailboxes)
	}()

	log.Println("Mailboxes:")
	for m := range mailboxes {
		log.Println("* " + m.Name)
	}

	if err := <-done; err != nil {
		log.Fatal(err)
	}

	// Select INBOX
	mbox, err := c.Select("INBOX", false)
	if err != nil {
		log.Fatal(err)
	}
	log.Println("Flags for INBOX:", mbox.Flags)

	// Get the last 4 messages
	from := uint32(1)
	to := mbox.Messages
	if mbox.Messages > 3 {
		// We're using unsigned integers here, only subtract if the result is > 0
		from = mbox.Messages - 3
	}
	seqset := new(imap.SeqSet)
	seqset.AddRange(from, to)

	messages := make(chan *imap.Message, 10)
	done = make(chan error, 1)
	go func() {
		done <- c.Fetch(seqset, []imap.FetchItem{imap.FetchEnvelope}, messages)
	}()

	log.Println("Last 4 messages:")
	for msg := range messages {
		log.Println("* " + msg.Envelope.Subject)
	}

	if err := <-done; err != nil {
		log.Fatal(err)
	}

	log.Println("Done!")
}
```

### Server [![godocs.io](https://godocs.io/github.com/ncastellani/imapServer/server?status.svg)](https://godocs.io/github.com/ncastellani/imapServer/server)

```go
package main

import (
	"log"

	"github.com/ncastellani/imapServer/server"
	"github.com/ncastellani/imapServer/backend/memory"
)

func main() {
	// Create a memory backend
	be := memory.New()

	// Create a new server
	s := server.New(be)
	s.Addr = ":1143"
	// Since we will use this server for testing only, we can allow plain text
	// authentication over unencrypted connections
	s.AllowInsecureAuth = true

	log.Println("Starting IMAP server at localhost:1143")
	if err := s.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

You can now use `telnet localhost 1143` to manually connect to the server.

## Extensions

Support for several IMAP extensions is included in go-imap itself. This
includes:

* [APPENDLIMIT](https://tools.ietf.org/html/rfc7889)
* [CHILDREN](https://tools.ietf.org/html/rfc3348)
* [ENABLE](https://tools.ietf.org/html/rfc5161)
* [IDLE](https://tools.ietf.org/html/rfc2177)
* [IMPORTANT](https://tools.ietf.org/html/rfc8457)
* [LITERAL+](https://tools.ietf.org/html/rfc7888)
* [MOVE](https://tools.ietf.org/html/rfc6851)
* [SASL-IR](https://tools.ietf.org/html/rfc4959)
* [SPECIAL-USE](https://tools.ietf.org/html/rfc6154)
* [UNSELECT](https://tools.ietf.org/html/rfc3691)

Support for other extensions is provided via separate packages. See below.

## Extending go-imap

### Extensions

Commands defined in IMAP extensions are available in other packages. See [the
wiki](https://github.com/ncastellani/imapServer/wiki/Using-extensions#using-client-extensions)
to learn how to use them.

* [COMPRESS](https://github.com/ncastellani/imapServer-compress)
* [ID](https://github.com/ProtonMail/go-imap-id)
* [METADATA](https://github.com/ncastellani/imapServer-metadata)
* [NAMESPACE](https://github.com/foxcpp/go-imap-namespace)
* [QUOTA](https://github.com/ncastellani/imapServer-quota)
* [SORT and THREAD](https://github.com/ncastellani/imapServer-sortthread)
* [UIDPLUS](https://github.com/ncastellani/imapServer-uidplus)

### Server backends

* [Memory](https://github.com/ncastellani/imapServer/tree/master/backend/memory) (for testing)
* [Multi](https://github.com/ncastellani/imapServer-multi)
* [PGP](https://github.com/ncastellani/imapServer-pgp)
* [Proxy](https://github.com/ncastellani/imapServer-proxy)
* [Notmuch](https://github.com/stbenjam/go-imap-notmuch) - Experimental gateway for [Notmuch](https://notmuchmail.org/)

### Related projects

* [go-message](https://github.com/emersion/go-message) - parsing and formatting MIME and mail messages
* [go-msgauth](https://github.com/emersion/go-msgauth) - handle DKIM, DMARC and Authentication-Results
* [go-pgpmail](https://github.com/emersion/go-pgpmail) - decrypting and encrypting mails with OpenPGP
* [go-sasl](https://github.com/emersion/go-sasl) - sending and receiving SASL authentications
* [go-smtp](https://github.com/emersion/go-smtp) - building SMTP clients and servers

## License

MIT
