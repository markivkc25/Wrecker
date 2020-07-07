<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:camel="http://camel.apache.org/schema/spring"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans-3.0.xsd        http://camel.apache.org/schema/spring https://camel.apache.org/schema/spring/camel-spring.xsd">
    <bean id="_response_processor" name="_response_processor"/>
    <camelContext id="camelContext-69db88bc-a15a-4842-aa59-45371ac3a5c4"
        trace="false" xmlns="http://camel.apache.org/schema/spring">
        <route errorHandlerRef="maOutboundErrorHandler"
            id="_outbound_edge_mediator" startupOrder="2">
            <description>Outbound Mediator which polls the file from out folder and sends to Local Processor for further processing</description>
            <from id="_fromMidasOutFileFolder" uri="file://fmacdata/pfdc/midas/out?fileName=PFDC*.csv&amp;exchangePattern=InOnly&amp;delay=30m&amp;initialDelay=30m&amp;timeUnit=MINUTES"/>
            <to id="_work_queue_edge_processor" uri="seda:work_queue"/>
        </route>
        <route errorHandlerRef="maOutboundErrorHandler"
            id="_outbound_error_reprocessor" startupOrder="2">
            <description>Outbound Mediator which polls the file from error folder and sends to Local Processor for further processing</description>
            <from id="_fromMidasErrorFileFolder" uri="file://fmacdata/pfdc/midas/error/out?fileName=PFDC*.csv&amp;exchangePattern=InOnly&amp;delay=30m&amp;initialDelay=30m&amp;timeUnit=MINUTES"/>
            <to id="_work_queue_error_processor" uri="seda:work_queue"/>
        </route>
        <route errorHandlerRef="maOutboundErrorHandler"
            id="_outbound_local_mediator" startupOrder="1">
            <description>Local Outbound Mediator which polls the local WorkQueue and sends to the pfdc event subscriber API</description>
            <from id="_work_queue" uri="seda:work_queue"/>
            <doTry id="FileProcessing">
                <bean id="_response_processor" method="parseCSV" ref="midasResponseTransformer"/>
                <bean id="_response_processor" method="prepareXML" ref="midasResponseTransformer"/>
                <to id="_pfdc_endpoint" uri="http://${OCP_URI}:/pfdc/api/v1/ldc-event/status"/>
                <doCatch id="FileProcessingException">
                    <!-- catch multiple exceptions -->
                    <exception>java.io.IOException</exception>
                    <exception>java.lang.IllegalStateException</exception>
                    <to id="_errorFolderEndpoint" uri="file://fmacdata/pfdc/midas/out/error"/>
                </doCatch>
                <doFinally id="_doFinally">
                    <to id="_auditFolderEndpoint" uri="file://fmacdata/pfdc/midas/out/audit?fileName=PFDC_${timeStamp}.csv"/>
                </doFinally>
            </doTry>
        </route>
    </camelContext>
</beans>
