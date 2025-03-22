# Provide your CLI command here:

## Objective
From a file `./transaction-log.txt`, identify orders selling `TSLA` and submit an HTTP GET request to `https://example.com/api/:order_id`. Save all responses into `./output.txt`.

---

## Solution: Using `jq` + `xargs` + `curl`

This is the most robust and efficient solution, relying on `jq` to safely parse JSON.

If you don't have `jp`, please 
```cmd
sudo apt install jp
```

```bash
jq -r 'select(.symbol == "TSLA" and .side == "sell") | .order_id' ./transaction-log.txt | xargs -I{} curl -s https://example.com/api/{} >> ./output.txt
```

## Explanation

This command performs a pipeline of operations to filter specific JSON entries and execute HTTP requests. Here's a breakdown of each component:

```bash
jq -r 'select(.symbol == "TSLA" and .side == "sell") | .order_id' ./transaction-log.txt
```
- `jq`: A lightweight and flexible command-line JSON processor.
- `'select(.symbol == "TSLA" and .side == "sell")'`: Filters only the entries where the symbol is `"TSLA"` and the side is `"sell"`.
- `| .order_id`: Extracts only the `order_id` field from those filtered entries.
- `-r`: Outputs raw strings instead of JSON-encoded strings.
- The result is a list of `order_id`s matching the criteria.

```bash
| xargs -I{} curl -s https://example.com/api/{}
```
- `xargs`: Reads items from standard input (the list of `order_id`s from `jq`) and executes the given command for each.
- `-I{}`: Defines a placeholder `{}` that will be replaced by each `order_id`.
- `curl -s https://example.com/api/{}`: Sends a silent (`-s`) HTTP GET request to the API endpoint for each order.

```bash
>> ./output.txt
```
- Appends (`>>`) the response of each `curl` command to the file `output.txt`.

---

## Summary

This one-liner:
1. Parses a log of JSON transactions,
2. Filters for sell orders of TSLA,
3. Extracts their order IDs,
4. Sends HTTP GET requests for each,
5. And saves the responses into `output.txt`.
