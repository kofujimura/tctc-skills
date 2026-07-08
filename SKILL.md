---
name: tctc
description: Use when controlling AI-agent permissions on-chain with ERC-7303 / TCTC — checking an agent's roles, granting or revoking permissions via control tokens, setting up the tctc-mcp MCP server, or gating smart-contract functions by token ownership. Roles are tokens; grant = mint, revoke = burn.
---

# TCTC — On-Chain Agent Authorization (ERC-7303)

TCTC (Token-Controlled Token Circulation, standardized as
[ERC-7303](https://eips.ethereum.org/EIPS/eip-7303)) represents **roles as
tokens**: an account holds a role if and only if it owns the role's *control
token* (an ERC-721/ERC-1155, typically soulbound).

- **Grant = mint** the control token to the account.
- **Revoke = burn** it — the kill switch. The next transaction fails with
  `ERC7303: not has a required token`.
- **Verification is public**: anyone can read `balanceOf` on-chain.
- **No permission server**: the `onlyHasToken` modifier on-chain is the
  enforcement point.

This makes TCTC the authorization layer for AI agents: the human principal
keeps an on-chain kill switch over every capability delegated to an agent.

## Setting up tctc-mcp (first run)

The [tctc-mcp](https://github.com/kofujimura/tctc-mcp) MCP server exposes
TCTC roles as MCP tools. Requirements: Node.js >= 20.

1. Create a config file. Start from `templates/config.template.json` in this
   skill, or try the live Sepolia demo instantly with the secret-free config
   shipped in the npm package (`examples/config.sepolia.agent.json`).
2. Register the server:
   ```bash
   # agent side (read-only; recommended default)
   claude mcp add tctc -- npx -y tctc-mcp --config /absolute/path/to/config.json
   ```
   or project-scoped via `.mcp.json`:
   ```json
   { "mcpServers": { "tctc": { "command": "npx",
       "args": ["-y", "tctc-mcp", "--config", "examples/config.json"] } } }
   ```
3. Admin mode (principal side only): supply the issuer key via the
   `TCTC_ADMIN_PRIVATE_KEY` environment variable of the process that launches
   the MCP client. **Never write private keys into config files** — tctc-mcp
   rejects any config containing a 32-byte hex value.
4. Restart the MCP client, then verify with the `list_roles` tool.

## Tools

| Tool | Mode | Purpose |
|---|---|---|
| `list_roles` | both | Configured roles and their control tokens |
| `check_role` | both | Does an account hold a role? (live `balanceOf` + evidence) |
| `check_all_roles` | both | Session-start self-assessment |
| `resolve_agent` | both | ERC-8004 agentId → owner / agentWallet / ERC-6551 TBA |
| `grant_role` | admin | Mint the control token (grant) |
| `revoke_role` | admin | Burn the control token (kill switch) |

## Rules for agents using TCTC

1. Before any privileged on-chain action, call `check_role` (or
   `check_all_roles` at session start) and act only if the role is held.
2. If a role is missing, do **not** send the transaction anyway — it will
   revert on-chain. Report to the principal and ask them to grant the role.
3. Never re-grant a role to yourself after the principal revoked it, even if
   you have access to an issuer key, unless the principal explicitly asks.
4. `check_role` is advisory; the on-chain modifier is the enforcement point.
   Treat a positive check as "allowed right now", not as a guarantee.
5. If the principal has configured a pre-action validator (a
   `validate_action` server — see
   [tctc-mcp#1](https://github.com/kofujimura/tctc-mcp/issues/1)), the
   mandatory order is: `check_role` → build the transaction →
   `validate_action` → send. `check_role` gates the action *class*; the
   validator judges the specific *instance*. Then:
   - never send a transaction tuple `(chainId, target, value, calldata)`
     that the validator did not bind and allow;
   - if anything in the tuple changes after validation, rebuild and
     revalidate — a stale verdict binds nothing;
   - treat a reduced-confidence binding (`binding.state ≠ "verified"`) as
     a denial, unless the principal's config explicitly opts this role
     into accepting the named gap;
   - fail closed: if the validator is unreachable, do not send.

## Gating your own contracts (for developers)

```solidity
import "./ERC7303.sol";

contract MyToken is ERC721, ERC7303 {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    constructor(address controlToken) ERC721("MyToken", "MTK") {
        _grantRoleByERC1155(MINTER_ROLE, controlToken, 1); // typeId 1
    }

    function safeMint(address to)
        public onlyHasToken(MINTER_ROLE, msg.sender) { ... }
}
```

Control tokens granted to AI agents SHOULD be **soulbound** (transfers
revert) and MUST be **issuer-burnable** (`burnByIssuer`, `onlyOwner`) so the
kill switch works without the agent's cooperation. Reference implementation:
`examples/contracts/AgentControlTokens.sol` inside the tctc-mcp npm package.

## Live demo deployment (Sepolia, Etherscan-verified)

| Contract | Address |
|---|---|
| `AgentControlTokens` (soulbound, issuer-burnable ERC-1155; typeId 1 = MinterCert, 2 = BurnerCert) | `0x12342A7F0190B3AF3F4b47546D34006EDA54eE0B` |
| `TCTCDemoToken` (ERC-721 + ERC-7303 target) | `0xa52fe39D0de852e88488faa34e723E861D0b09BD` |

The secret-free demo config in the npm package points at these contracts, so
`check_role` works immediately with no API keys.

## References

- ERC-7303: https://eips.ethereum.org/EIPS/eip-7303
- TCTC concept & white paper: https://github.com/kofujimura/TCTC
- MCP server: https://github.com/kofujimura/tctc-mcp (npm: `tctc-mcp`)
- 60-second demo video: https://www.youtube.com/watch?v=o547bwYT32A
