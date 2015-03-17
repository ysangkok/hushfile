Minimal API examples
===

hushfile-web can individually encrypt chunks and [PUT at /](http://ysangkok.github.io/swagger-ui/dist/index.html?url=/hushfile/minimal-v1.yaml#!/default/_put).

Say the encrypted chunks (they could of course be arbitrary bytes in a real scenario) are as follows:

* chunk 1: `ABCD`,
* chunk 2: `0123456789`,
* chunk 3: `xy`

After all chunks have been uploaded with deletekeys (`lol` for chunk 1 `bob` for chunk 2, `dan` for chunk 3) and no expiry, maybe they were assigned the Hushfile IDs:

* `absauidsahui` (chunk 1)
* `bsjioasjidao` (chunk 2)
* `ccjsakasiojd` (chunk 3)

The client has the ID's, and their deletekeys. Hushfile-web can generate a TOC-and-metadata file:

```
{
	"metadata": "jsaiodcnsuiovdshdsiuovhs ENCRYPTED METADATA (base64) dsjiuoadjsaiodj",
	"chunklengths": [
		4,
		10,
		2
	]
}
```

And upload it as it's own hushfile. This is assigned ID `DWJQIO` and the client chose the deletekey `jan`.

The client now makes a request to [`POST /merge`](http://ysangkok.github.io/swagger-ui/dist/index.html?url=/hushfile/minimal-v1.yaml#!/default/merge_post) that looks roughly like this:

```
POST /api/merge HTTP/1.1
Host: hushfile.it
Content-Type: application/x-www-form-urlencoded

id=DWJQIO&id=absauidsahui&id=bsjioasjidao&id=ccjsakasiojd&deletekey=d5
```

(the client could have specified a expiry date, but chose not to)

On the server, the data could have been stores like this:
```
.
├── [       4120]  absauidsahui
│   ├── [          4]  data
│   └── [         20]  serverdata
├── [       4126]  bsjioasjidao
│   ├── [         10]  data
│   └── [         20]  serverdata
├── [       4118]  ccjsakasiojd
│   ├── [          2]  data
│   └── [         20]  serverdata
└── [       4242]  DWJQIO
    ├── [        126]  data
    └── [         20]  serverdata

       20702 bytes used in 4 directories, 8 files
```

The server can save uploader IP's in an array in the individual serverdata files.

Since a hushfile can't be edited, the merge can simply make a new directory and hardlink (also on windows) the data chunks into the new directory. Lets say the new hushfile has the ID `POIUYT`. The server could do this:

```
ln DWJQIO/data.0 POIUYT/data.0
ln absauidsahui/data.0 POIUYT/data.1
ln bsjioasjidao/data.0 POIUYT/data.2
ln ccjsakasiojd/data.0 POIUYT/data.3
```

and make a new serverdata file, maybe copying the uploader IP's from the individual chunks. The expiry date would be set anew, but this isn't really a problem since with any API that allows anonymous reads, the expiry won't be very effective.

Say a client gets a link to `hushfile.it/POIUYT`. Hushfile-web will send a request for the first 100 bytes and see if it has a complete JSON object (the data from the file uploaded as DWJQIO) yet. When feeding the bytes into [oboejs](http://oboejs.com/) or [clarinet](https://github.com/dscape/clarinet), it doesn't matter that there are extranous binary data after the TOC-and-metadata, cause a SAX style parser will fire the event for the object before getting syntax errors. If the 100 bytes are not enough, the client keeps requesting more until it finds a complete object.

Now that the client has the TOC, it also knows the offset for the first chunk, since the first chunk starts immediatly after the TOC. It also knows the length of the chunk, since that was encoded into the TOC. The client can now, in parallel, download and decrypt all chunks.
