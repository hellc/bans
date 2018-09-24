# Simple BANS (Blockchain Address Name Service)
### DNS based Simple Blockchain Address retrieval method

The purpose of this method is to provide ability to link
blockchain wallet or smart contract addresses with existing public domain names.

![demo](https://github.com/hellc/bans/blob/master/demo/bans.gif)

## SETUP GUIDE

The main goal is to provide secured and anonymous features to link and retrieve your custom blockchain address based on public domain name you own. That will allow you to use short and user friendly names that could be found by everyone 
throught the simple validation procedure.

### STEPS

#### 0. You will have to use your existing blockchain address in order to link it with your own domain name.
#### 1. Select private and secured DNS.

We recommend you to select DNS with DNSSEC and DNS over HTTPS/TLS support.
For example: Cloudflare DNS service should be good enought. Also you could take a look at Google DNS.
Cloudflare as well provide TOR based DNS services. Check: https://blog.cloudflare.com/welcome-hidden-resolver/

#### 2. Create BANS DNS record

There are no specification on bans record format, however we are suggest you to follow:

```
DNS record
type: TXT
name: bans.<token code>.<code address seq. number>
value: <your blockchain address>
```
For example: 

```
DNS record 1
type: TXT
name: bans.xvg.0
value: DCfgkLJXMMAjcnKxTuw3CaTqNmgT7mwc5T

DNS record 2
type: TXT
name: bans.xvg.1
value: DCfgkLJXMMAjcnKxTuw3CaTqNmgT7mwc5A

DNS record 3
type: TXT
name: bans.btc.0
value: 1A8BS8UTKq7PMunsHKFPgzHq122gZ3vkqS

...
```
Result Cloudflare record should looks like:

![demo](https://github.com/hellc/bans/blob/master/demo/dns_Record.png)

#### 3. Retrieve your DNS record on client

To demonstrate this we will [fork](https://github.com/hellc/vIOS) the upcoming [XVG iOS wallet](https://github.com/vergecurrency/vIOS). 
To provide the anonymity we will perform connection via __TOR__ network using __DNS over HTTPS__ JSON request method.
_KXP.ONE_ - domain name will be used for our demonstration.

SWIFT base method will look like:
```SWIFT
class CloudflareAPIClient {
    
    static let shared = CloudflareAPIClient()
    
    let torEndpoint: String = "https://dns4torpnlfs2ifuz2s2yf3fc7rdmsbhm6rw75euj35pac6ap25zgqad.onion/dns-query?"
    let basicEndpoint: String = "https://cloudflare-dns.com/dns-query?"
    
    func walletAddressFor(currency: String, domainName: String, torDns: Bool = true,
                          completion: @escaping (_ address: String?) -> Void) {
        let endpoint = torDns ? torEndpoint : basicEndpoint
        let url = URL(string: "\(endpoint)name=bans.\(currency).0.\(domainName)&type=TXT")
        
        var request = URLRequest(url: url!)
        request.setValue("application/dns-json", forHTTPHeaderField: "accept")
        
        let task = TorClient.shared.session.dataTask(with: request){ (data, resonse, error) in
            if let data = data {
                do {
                    let json = try JSON(data: data)
                    let result = json["Answer"][0]["data"].string?.replacingOccurrences(of: "\"", with: "")
                    
                    completion(result)
                } catch {
                    print("Error info: \(error)")
                    completion(nil)
                }
            } else if let _ = error {
                completion(nil)
            }
        }
        
        task.resume()
    }
}
```
Here the request:

```SWIFT
CloudflareAPIClient.shared.walletAddressFor(currency: "xvg", domainName: "kxp.one") { 
                resultAddress in
                
                if resultAddress?.count != 34 { //SIMPLE ADDRESS VALIDATION
                    //DO: ADDRESS IS NOT VALID
                    return
                }
                //DO: ADDRESS VALID, PROCEED ADDRESS
                //resultAddress: "DCfgkLJXMMAjcnKxTuw3CaTqNmgT7mwc5T"
            }
```

Same via curl:

```curl
curl -H 'accept: application/dns-json' 'https://cloudflare-dns.com/dns-query?name=bans.xvg.0.kxp.one&type=TXT'
or
curl -H 'accept: application/dns-json' 'https://dns4torpnlfs2ifuz2s2yf3fc7rdmsbhm6rw75euj35pac6ap25zgqad.onion/dns-query?name=bans.xvg.0.kxp.one&type=TXT'

output:
{"Status": 0,"TC": false,"RD": true, "RA": true, "AD": false,"CD": false,"Question":[{"name": "bans.xvg.0.kxp.one.", "type": 16}],"Answer":[{"name": "bans.xvg.0.kxp.one.", "type": 16, "TTL": 300, "data": "\"DCfgkLJXMMAjcnKxTuw3CaTqNmgT7mwc5T\""}]}
```

To provide additional layer of security - perform your request using __DO__ field to get DNSSEC feature.
For more info, please visit: https://developers.cloudflare.com/1.1.1.1/dns-over-https/json-format/
