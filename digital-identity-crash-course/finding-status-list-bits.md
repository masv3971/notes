# Finding: the verifier ignores `bits` in Token Status Lists

Repository: `SUNET/vc`. Checked 2026-11-18 by code search on the default branch. Not run, not confirmed by the team.

## Summary

The revocation checker in `pkg/revocation` decompresses `lst` and reads one byte at position `idx`. It does not read `bits`. This is correct for our own lists, which are always 8 bits per entry. A list from another issuer that uses 1, 2 or 4 bits would be read at the wrong position, with no error.

The IETF draft (draft-ietf-oauth-status-list-21, section 4.1) allows `bits` of 1, 2, 4 or 8. It packs `8/bits` entries into each byte, starting from the least significant bit. A verifier has to read `bits` and unpack the right entry.

## What was found

| Where | What it does |
|---|---|
| `pkg/tokenstatuslist/types.go` | `const Bits = 8`. The comment says "we hardcode to 8 bits (1 byte per status)". |
| `pkg/tokenstatuslist/jwt.go` | The writer sets `Bits: Bits`, so every token it produces says `"bits": 8`. |
| `pkg/tokenstatuslist/types.go` | `DecompressStatuses` returns the decompressed bytes as `[]uint8`, one value per byte. |
| `pkg/revocation/status_list.go` | `tok.statuses, err = tokenstatuslist.DecompressStatuses(lst)`, then `statusCode := statuses[ref.Index]`. An out-of-range index is rejected. |
| `pkg/revocation/extract.go` | Reads `idx` from the credential's `status_list` claim. |
| `developer_tools/scripts/tsl_checker/main.go` | `statusAt(lst, bits, idx)` handles `bits`: shift is `(idx % entriesPerByte) * bits`, mask is `(1 << bits) - 1`. |

A code search for `bits` in `pkg/revocation` found nothing that reads it. The only matches are a test that builds a token with `Bits: 8` and the CWT struct in `pkg/tokenstatuslist`, which parses it.

## Effect

- Our own lists work, because the reader assumes 8 bits and the writer always uses 8.
- A foreign list with `bits` of 1, 2 or 4 gives wrong answers. For example, with `bits: 1`, entry 9 is in bit 1 of byte 1, but the reader returns the whole of byte 9. A revoked credential can then look valid, or a valid one revoked.
- There is no error. The out-of-range check compares `idx` against the number of bytes, not the number of entries.
- Risk today is low while all lists come from SUNET/vc. It grows when verifying credentials from other issuers, which is the stated direction.

## Suggested fix

1. Read `bits` from the status list (JWT `status_list.bits`, CWT key 1).
2. Accept only 1, 2, 4 and 8. Reject anything else.
3. Unpack the entry the same way `statusAt` does in `tsl_checker`, and check `idx` against the number of entries, not bytes.
4. Add test vectors with `bits` of 1, 2 and 4. The draft's Appendix C has them.
5. Consider moving `statusAt` out of the CLI tool into `pkg/tokenstatuslist`, so the verifier and the tool share one implementation.

## Not checked

- Only search excerpts were read, not the whole of `status_list.go`. A `bits` check could exist in a part not seen.
- The apigw and issuing login path (see the open items in handout Appendix B) were not checked.
- The CWT path (`GetStatusFromCWT` in `pkg/tokenstatuslist/cwt.go`) also decompresses and indexes. Whether it reads `bits` was not confirmed.
- Nothing was run. No test with `bits` other than 8 was tried.

## Related

- Handout Appendix B, "Four things are still open" (`bits` is ignored).
- Handout Appendix B, CWT claim keys: the code uses 65534 and 65535, where draft -21 requests 65533, 65534 and 65535.
