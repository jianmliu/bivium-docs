# MCP for agents

Connect an MCP-compatible agent to Bivium to explore markets, preview a strategy and prepare an unsigned transaction. You review the result and execute it with your own wallet. The MCP server does not hold private keys, sign transactions or broadcast them.

The hosted service targets **Robinhood testnet (chain 46630)**. It is a test deployment, not a mainnet trading service.

## Connect your agent

| Setting | Value |
| --- | --- |
| Endpoint | `https://bivium-mcp.liujm06.workers.dev/mcp` |
| Transport | Streamable HTTP |
| Authentication | `Authorization: Bearer YOUR_ACCESS_TOKEN` |
| Risk policy | `conservative` |

Obtain an access token from the deployment operator. Use an MCP client that supports HTTP endpoints and custom Authorization headers. Automatic OAuth login is not available; a client that only supports OAuth cannot connect directly to this service.

For clients using an `mcpServers` configuration, add:

```json
{
  "mcpServers": {
    "bivium": {
      "type": "http",
      "url": "https://bivium-mcp.liujm06.workers.dev/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_ACCESS_TOKEN"
      }
    }
  }
}
```

Replace `YOUR_ACCESS_TOKEN` locally. Configuration keys can differ between clients; the endpoint, transport and Authorization header are the same. Keep the token in your client's private configuration or secret store, not in a URL, prompt, tool argument or source repository. It is a service access token, not a wallet key.

After connecting, use `server_info` to confirm the chain, core contract, policy IDs and execution capabilities. Let your client discover current input schemas through MCP `tools/list`.

## Four initial strategies

| Strategy ID | Intent | `size` in human token units | `maxInput` in human token units |
| --- | --- | --- | --- |
| `lendAsset` | Lend the asset through its DCN market | Asset DCN face to acquire | Maximum asset spend, including lender fee |
| `lendQuote` | Lend the quote token through the selected market | Quote-token DCN face to acquire | Maximum quote-token spend, including lender fee |
| `short` | Borrow the asset and sell it | Borrowed asset face | Maximum quote collateral drawn from the wallet after the swap |
| `leveredLong` | Borrow against an asset holding and buy more of the asset | Asset holding used to size borrowing at the selected strike | Maximum asset collateral drawn from the wallet after the swap |

`size` is not a spending budget. For `leveredLong`, it is a sizing basis, not a leverage multiplier. `maxInput` bounds the wallet contribution; it does not cap all losses on existing inventory or guarantee profit.

Strategy availability depends on current order-book depth, supported routing, fees, account state and risk evidence. A listed strategy is not a promise that an executable order exists.

## From strategy to wallet

1. **Discover.** Call `strategy_list` for capabilities and `market_list` to select an actually listed maturity. Use `market_details` and `book_snapshot` to inspect the selected market. Similar token symbols do not establish the same market: verify the chain, core, token addresses, maturity, strike and gate.
2. **Preview.** Call `strategy_preview` with the user's intended strategy, amount and bounds, plus `policyId: "conservative"` and current collateral evidence. Review the selected market, exact fees, wallet input, swap bounds, warnings and risk report. `strategy_quote` is indicative; its `quoteId` cannot substitute for a `previewId`.
3. **Prepare.** Call `action_prepare` with the returned `previewId` in the **same MCP session**. The server revalidates the preview and returns an unsigned transaction or prerequisite approvals.
4. **Execute with your wallet.** Review and submit prerequisite approvals externally if required. Check their receipts with `transaction_status`, then repeat `strategy_preview` using the original high-level request before preparing again. When ready, sign and submit the prepared transaction with your wallet. Verify its receipt and resulting `account_snapshot` or `strategy_positions`.

All four previews require `strategy`, `asset`, `size`, `maturity`, `bufferPct`, `account`, `maxInput`, `slippageBps`, `policyId`, `collateralKind` and `evidence`. `short` and `leveredLong` also require `maxPriceImpactBps`. Amounts such as `size` and `maxInput` are decimal strings; `maturity` is a Unix timestamp string. Consult the discovered schema for types and limits on every field.

The server resolves executable fills and routing from current data. Do not invent a router, pool key or fill to work around a rejected preview. Do not widen the user's limits merely to obtain a passing result.

## Risk and execution boundaries

The hosted `conservative` policy requires confirmation when risk evidence is missing and rejects some collateral, including arbitrary-mint or unsellable collateral. A test token being mintable does not make it acceptable under this policy. A policy rejection stops preparation; a confirmation result is not approval to execute.

No full-size executable order, insufficient depth, a stale source, unsupported routing or excessive price impact can block a preview. An unavailable source is not an empty book or a zero fee. There is no automatic fallback to placing a resting order.

The remote service does not publish or delist relayer orders. `order_publish` and `order_delist` are discoverable for compatibility but refuse writes on this deployment. Signing and transaction broadcasting always happen outside the MCP server.

## Sessions and troubleshooting

Your MCP client handles initialization, protocol headers and session IDs. This endpoint implements MCP 2025-06-18 with JSON responses. It requires requests to accept both `application/json` and `text/event-stream`; GET `/mcp` returns 405 because there is no standalone SSE stream. Use an MCP client rather than opening the endpoint as a webpage.

| Result | Next step |
| --- | --- |
| HTTP 401 | Check the Bearer token and Authorization header in the client configuration. |
| HTTP 403 | A browser Origin or host was rejected. Use a supported native client or ask the operator to configure the intended browser origin. |
| HTTP 404 for an existing session | Initialize a new session and obtain a fresh preview. |
| HTTP 429 | Back off; the service has session and request limits. |
| Expired preview | Run `strategy_preview` again before `action_prepare`. |
| Prerequisite approvals returned | Review and execute them in the wallet, confirm receipts, then repreview. |
| Risk rejection or missing liquidity | Inspect the report or book. Do not relax limits or reinterpret missing data as permission. |

Sessions expire after 15 minutes idle and may end earlier when Cloudflare restarts or evicts the coordinator. Preview lifetimes are at most 60 seconds and preview IDs cannot be transferred between sessions. Prepared state is ephemeral; it is not a persistent execution queue.

A public [health endpoint](https://bivium-mcp.liujm06.workers.dev/health) reports transport liveness. It does not establish that a market has executable liquidity or that an upstream RPC is healthy.

## Run your own server

For local stdio, a local HTTP listener, authentication setup and Cloudflare deployment instructions, see the [CLI MCP transport guide](https://github.com/jianmliu/bivium-cli/blob/main/docs/mcp-http.md). For detailed amounts, policy semantics, maker workflows and cancellation, see the [agent workflow reference](https://github.com/jianmliu/bivium-cli/blob/main/skills/bivium/references/mcp.md).

The [command-line interface](cli.md) remains available for direct scripting and local wallet operations.
