+++
title = "A Celestia Blob Is a Receipt, Not a Copy of the Payslip"
description = "How I want a Celestia blob to seal a SigmaUno Personale payslip: publish a hash, keep the file, and store the inclusion proof while the blob can still be served."
draft = true

[taxonomies]
tags = ["Celestia", "DevOps"]
+++

[Personale](https://sigmapersonale.com) generates payslips. Once one has been handed to someone, I need to be able to show that the file I still have is the file I published. A row in the Personale database doesn't do that. I can update a row. I can regenerate a PDF under the same name. A Celestia blob is the receipt I use instead: the hash of the document was published in this namespace, at this height, and the header still commits to it.

I work on Celestia's custom chains team. This is not that work. It is SigmaUno using the public network for a document Personale gives to a person. Personale has been live with its first customer since 16 November 2026. Nothing below is about their payroll.

<!-- more -->

## The blob does not hold the payslip

A Celestia blob is public. A payslip is full of data that should not be. The blob carries a hash of the canonical bytes, plus an id that does not identify a person. Personale keeps the file. If I lose the file, the hash does not bring it back. The blob is a seal. The archive stays in Personale.

Two hashes are involved, and they cover different bytes.

1. SHA-256 of the canonical document. That digest goes in the payload.
2. The blob commitment over the exact payload bytes. That commitment is what the PayForBlobs transaction publishes, and what the header's data root eventually commits to.

Changing the PDF fails the first check. Editing the stored payload so that it matches a new PDF fails the second, because the commitment in the header was over the old bytes. A later blob would be a new publication, with a new height, which is a different event from quietly replacing the file.

**TODO(alex):** what the canonical bytes are. The PDF, a canonical JSON document, or both, and the rules: field order, timestamps, whether a signature block is inside the hash.

## The payload

This struct is a stand-in so the rest of the post has something to point at. It is not a claim that Personale posts this JSON.

```go
type payslipReceipt struct {
    Doc    string `json:"doc"`    // "payslip"
    ID     string `json:"id"`     // opaque
    SHA256 string `json:"sha256"` // hex digest of the canonical document
    Issued string `json:"issued"` // RFC3339
}
```

The id stays opaque. A name, a tax code, or an email in a public blob is a leak on its own, with no salary attached.

Store the exact payload bytes that were submitted, and check against those. The commitment covers that encoding. A later `json.Marshal` of the same struct happens to be stable in today's `encoding/json` for a struct of strings, until the field set or the encoder changes, and the encoder escapes `&`, `<`, and `>`. "The same object" is not "the same bytes."

**TODO(alex):** replace the struct with the fields you actually post. Confirm that none of them identify a person.

## Posting it

The call shape below is the current [celestia-node Go client](https://docs.celestia.org/build/post-retrieve-blob/client/go/): `share.NewV0Namespace`, `blob.NewBlob` at share version 0, then `Blob.Submit`, which returns the height. The commitment is on the blob before the submit. This is the tutorial's shape, not a paste of the Personale poster. The import paths have moved before.

```go
ns, err := share.NewV0Namespace(namespaceID)
if err != nil {
    return err
}
payload, err := json.Marshal(record) // stand-in; store these exact bytes
if err != nil {
    return err
}
b, err := blob.NewBlob(share.ShareVersionZero, ns, payload, nil)
if err != nil {
    return err
}
height, err := client.Blob.Submit(ctx, []*blob.Blob{b}, nil)
if err != nil {
    return err
}
```

What gets written next to the payslip, in Personale:

- the document
- the exact payload bytes
- the namespace
- the height
- `b.Commitment`
- an inclusion proof, fetched while some node will still serve the blob

**TODO(alex):** the namespace bytes, and whether this posts to Mocha or to Mainnet.

**TODO(alex):** the module versions you build against, if the import paths above are not the ones in the Personale poster.

**TODO(alex):** whether the poster talks to a node you run or to a hosted DA RPC. I don't have an endpoint to put here, and I don't want a guessed one.

## The blob is not a disk you rent

Celestia's job is to order the blobs and to make them available when they are published, with that commitment in the header. [The docs](https://docs.celestia.org/learn/celestia-101/retrievability/) say the next part plainly: once the data is published, the network does not promise that the historical bytes stay retrievable. As of celestia-app v6, light nodes sample a 7-day window (CIP-36). Blobs older than that window are pruned on those nodes. An archival node may still have them. Personale does not get to depend on a stranger continuing to serve the namespace for free.

So a check I want to run next year does not start with `Blob.Get`. That needs someone to still be serving the shares. `Blob.Included`, as celestia-node's blob service is written today, re-fetches the shares and recomputes the commitment before it compares proofs. Same limit. Use it while the blob is fresh. Once you are past the sampling window, and you have decided not to depend on an archival node, a proof you stored at publish time is the check that still runs.

The check that survives pruning uses only what Personale stored, plus the header chain, which consensus keeps:

1. Hash the document you have now. It has to equal `sha256` in the stored payload.
2. Recompute the blob commitment from the stored payload bytes, the namespace, and the share version. It has to equal the commitment you wrote down.
3. Verify the inclusion proof you stored at publish time against the data root in the Celestia header at that height.

`GetProof` gives an NMT proof of the blob in the rows it spans. `GetCommitmentProof` is the one that carries the commitment up to the data root. Step 3 wants the second one, or whatever you have that a verifier can check without downloading shares.

**TODO(alex):** which proof the poster actually stores. If it stores neither, the post should describe the check you can still run, including the point at which it starts depending on an archival node.

**TODO(alex):** who is allowed to run the check. The employee, the customer, an auditor, or only us, and whether Personale exposes it or it's a script. I have not described a button.

## One document at a time, until that stops being true

With one customer I can post one receipt per payslip and stop. A Merkle root of a batch is the obvious change when one blob per document stops being worth the fee. The root goes in the blob. Each payslip keeps its own Merkle proof into that root, plus the one Celestia inclusion proof for the batch. I'm not putting a batch size or a fee in this post.

**TODO(alex):** whether you already batch, and what the batch is. A pay run, a day, a customer. "We don't" is a fine answer.

## What the receipt establishes

It establishes that these document bytes were the ones committed to at that height, under that namespace. It does not establish that the payroll math was right. A hash of a wrong net pay is a perfectly valid hash. The figures belong to the calculation that produced the file.

It also establishes, to anyone who can read the namespace, that a document with that opaque id was published then. That's what a public receipt is. It's why the payload stays small.
