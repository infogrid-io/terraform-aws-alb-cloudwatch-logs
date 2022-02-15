# terraform-aws-alb-cloudwatch-logs

This Terraform module ships AWS ALB logs to CloudWatch Logs.

## Requirements

* Terraform 0.12.x or above
* Python

## Example

```tf
resource "aws_cloudwatch_log_group" "test" {
  name              = aws_alb.test.name
  retention_in_days = 365
}

module "alb_logs_to_cloudwatch" {
  source  = "terraform-aws-alb-cloudwatch-logs"
  version = "1.0.0"

  bucket_name    = aws_s3_bucket.logs.bucket
  log_group_name = aws_cloudwatch_log_group.test.name

  memory_size = 256
  timeout     = 30

  create_alarm  = true
  alarm_actions = [aws_sns_topic.slack.arn]
  ok_actions    = [aws_sns_topic.slack.arn]
}

resource "aws_lambda_permission" "bucket" {
  statement_id  = "AllowExecutionFromS3Bucket"
  action        = "lambda:InvokeFunction"
  function_name = module.alb_logs_to_cloudwatch.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.logs.arn
}

resource "aws_s3_bucket_notification" "logs" {
  bucket     = aws_s3_bucket.logs.bucket
  depends_on = ["aws_lambda_permission.bucket"]

  lambda_function {
    lambda_function_arn = module.alb_logs_to_cloudwatch.function_arn
    events              = ["s3:ObjectCreated:*"]
  }
}
```

## Using CloudWatch Logs Insights

When using `format_json = false` (the default), the following query may be useful for getting started. The `parse` query will parse the log message into individual fields.

```
fields @timestamp, @message
| parse '* * * * * * * * * * * * "* * *" "*" * * * "*" "*" "*" * * "*" "*" "*" "*" "*" "*" "*"' as type, time, elb, client, target, request_processing_time, target_processing_time, response_processing_time, elb_status_code, target_status_code, received_bytes, sent_bytes, request_verb, request_url, request_proto, user_agent, ssl_cipher, ssl_protocol, target_group_arn, trace_id, domain_name, chosen_cert_arn, matched_rule_priority, request_creation_time, actions_executed, redirect_url, error_reason, target_port_list, target_status_code_list, classification, classification_reason
| sort @timestamp desc
| limit 20
| display request_url, elb_status_code
```

When using this module with `format_json = true`, using Logs Insights is easier. The log messages will be JSON objects, so the fields can be used without a `parse` query. This can drastically increase the size and cost of the logs (perhaps triple, depending on URL lengths), and the `parse` query is easy enough to copy/paste, so JSON formatting is disabled by default.
