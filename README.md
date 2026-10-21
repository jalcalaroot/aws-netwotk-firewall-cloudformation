# Centralized Egress Inspection with AWS Network Firewall (CloudFormation)

CloudFormation templates for a hub-and-spoke network: a Transit Gateway, an inspection VPC with AWS
Network Firewall and NAT Gateways (2 or 3 AZs), a 4-layer application VPC and a Network Firewall
CloudWatch dashboard.

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

```
spoke VPCs --TGW--> inspection VPC: TGW subnet -> Network Firewall endpoint -> NAT subnet (NAT GW) -> IGW
```

- `TransitGateway-latest.yaml`: Transit Gateway (ASN 64512, default association/propagation disabled,
  auto-accept shared attachments) and one route table (`<stack>-tgw-main-rt`).
- `Network-Firewall-2azs-latest.yaml` / `Network-Firewall-3azs-latest.yaml`: inspection VPC with IGW,
  TGW/firewall/NAT subnets per AZ (carved with `!Cidr`), open NACLs, one NAT Gateway + EIP per AZ,
  Network Firewall with an empty policy (everything forwarded to the stateful engine, no rule groups),
  FLOW/ALERT/TLS logs to CloudWatch, a TGW attachment associated with the given TGW route table and a
  `0.0.0.0/0` TGW route to it. Return routes to the spokes are hardcoded: `10.120.0.0/16` (2 AZ) and
  `192.168.0.0/16` (3 AZ).
- `application-vpc-latest.yml`: spoke VPC (default `192.168.16.0/20`) with public, compute, data and
  transit subnets in 3 AZs, IGW route for the public tier only, NACLs denying inbound SSH/RDP from the
  internet, S3/DynamoDB gateway endpoints, SSM/SSM Messages/API Gateway interface endpoints, VPC Flow
  Logs to CloudWatch, and exported IDs. It does **not** create a TGW attachment or TGW routes.
- `nfw-cloudwatch-dashboard.yaml`: CloudWatch dashboard plus Contributor Insights rules over the
  firewall flow/alert logs, with a small Lambda custom resource that builds the subnet query string.
- Diagrams: `AWS Accounts Design - Networking Model  vNext.png` (target multi-account design),
  `applicatios-vpc-config.png`, `Global_CIDR.png` (192.168.0.0/16 split into /20s) and the AWS
  reference PDF `inspection-deployment-models-with-AWS-network-firewall-ra.pdf`.

## Parameters

| Template | Name | Purpose | Default |
|----------|------|---------|---------|
| Inspection | `Name` | Tag/name prefix | `Inspection` |
| Inspection | `Region` | Region used to build AZ names (`<Region>a/b/c`); must match the deploy region | `us-east-1` |
| Inspection | `TransitGatewayId` / `TransitGatewayMainRouteTableId` | Outputs of the TGW stack | `<TGW_ID>` / `<TGW_ROUTE_TABLE_ID>` |
| Inspection | `VPCCIDR` | Inspection VPC CIDR | `10.120.16.0/20` (2 AZ), `172.16.0.0/18` (3 AZ) |
| Application | `VPCCIDR` and 12 `Subnet*CidrBlock` | VPC and per-tier subnet CIDRs | `192.168.16.0/20` layout |
| Application | `VpcName` | Name prefix / tenant ID, used in export names | - |
| Application | `LogRetention` | Flow log retention (days) | `30` |
| Dashboard | `FirewallName`, `FirewallSubnetList` | Firewall name (`<Name>-NFW`) and its endpoint subnets | - |
| Dashboard | `FirewallFlowLogGroupName`, `FirewallAlertLogGroupName` | `/<stack>/inspection-fw/flow` and `/alert` | - |
| Dashboard | `ContributorInsightsRuleState` | `ENABLED`/`DISABLED` | `ENABLED` |

## Deploy

Order: Transit Gateway -> inspection VPC (2 or 3 AZ) -> application VPC(s) -> dashboard. Adjust the hardcoded
spoke summary route in the inspection template to your CIDR plan first.

```bash
aws cloudformation deploy --stack-name tgw --template-file TransitGateway-latest.yaml
# TGW route table ID: aws ec2 describe-transit-gateway-route-tables (output only exposes the TGW ID)
aws cloudformation deploy --stack-name inspection --template-file Network-Firewall-3azs-latest.yaml \
  --parameter-overrides Region=<REGION> TransitGatewayId=<TGW_ID> TransitGatewayMainRouteTableId=<TGW_ROUTE_TABLE_ID>
aws cloudformation deploy --stack-name app-vpc --template-file application-vpc-latest.yml \
  --capabilities CAPABILITY_NAMED_IAM --parameter-overrides VpcName=<VPC_NAME>
aws cloudformation deploy --stack-name nfw-dashboard --template-file nfw-cloudwatch-dashboard.yaml \
  --capabilities CAPABILITY_IAM --parameter-overrides FirewallName=Inspection-NFW \
  FirewallSubnetList=<NFW_SUBNET_A>,<NFW_SUBNET_B>,<NFW_SUBNET_C> \
  FirewallFlowLogGroupName=/inspection/inspection-fw/flow FirewallAlertLogGroupName=/inspection/inspection-fw/alert
```

Spoke attachments, their TGW route table associations and the spoke `0.0.0.0/0 -> TGW` routes are manual.

## Security notes

- **The firewall policy has no rule groups**: traffic is logged but allowed (default stateful pass).
- The TGW auto-accepts shared attachments; review RAM sharing before using it.
- Inspection VPC NACLs allow all traffic; the TGW attachment does not enable appliance mode (possible
  asymmetric routing across AZs).
- The application VPC endpoint security group allows 443 from `0.0.0.0/0` (VPC-internal in practice).
- Firewall log groups have no retention or KMS encryption; dashboard Lambda uses `python3.9` (deprecated).

## Cleanup

Delete in reverse order: `nfw-dashboard`, `app-vpc`, `inspection`, then `tgw` (`aws cloudformation
delete-stack --stack-name <STACK>`). Remove any manually created spoke attachments first. NAT Gateways
and the firewall endpoints are billed hourly while the inspection stack exists.

## License

MIT — see [LICENSE](LICENSE).
