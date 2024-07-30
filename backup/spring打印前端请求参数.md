package com.gz.property.config;

import cn.hutool.json.JSONUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import javax.servlet.FilterChain;
import javax.servlet.ReadListener;
import javax.servlet.ServletException;
import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import javax.servlet.http.HttpServletResponse;
import java.io.*;
import java.util.Map;
import java.util.Objects;

/**
 * 打印请求信息拦截器
 * 为减少性能损耗，可使用isdebug = false 或者 注释此拦截器类。此类无任何侵入。独立类
 */
@Slf4j
@Configuration
@ConditionalOnProperty(name = "isdebug", havingValue = "true", matchIfMissing = true)
public class ApiInterceptor extends OncePerRequestFilter implements HandlerInterceptor, WebMvcConfigurer {
    /**
     * 注册自定义过滤器
     * @return
     */
    @Bean
    public FilterRegistrationBean<ApiInterceptor> addFilter() {
        FilterRegistrationBean<ApiInterceptor> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(this);
        registrationBean.addUrlPatterns("/*"); // 适用所有URL模式
        registrationBean.setOrder(1); // 设置过滤器的顺序，数字越小越优先执行
        return registrationBean;
    }

    /**
     * 注册自定义拦截器
     * @param registry
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(this);
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        //使用RequestWrapper包装request解决request.getBody()只能读取一次的问题
        RequestWrapper requestWrapper = null;
        try {
            requestWrapper = new RequestWrapper(request);
        }catch (Exception e){
            log.warn("customHttpServletRequestWrapper Error:", e);
        }

        filterChain.doFilter((Objects.isNull(requestWrapper) ? request : requestWrapper), response);
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (request instanceof RequestWrapper) {
            String requestURI = request.getRequestURI();
            String method = request.getMethod();
            Map<String, String[]> params = request.getParameterMap();
            if ("GET".equals(method)) {
                log.warn("Request URI: {}, Method: {}, Params: {}", requestURI, method, JSONUtil.parse(params));
            } else {
                log.warn("Request URI: {}, Method: {}, Params: {}, Body: {}", requestURI, method, JSONUtil.parse(params), JSONUtil.parse(((RequestWrapper) request).getBody()));
            }
        }
        return true;
    }
    private static class RequestWrapper extends HttpServletRequestWrapper {
        // 保存request body的数据
        private String body;

        // 解析request的inputStream(即body)数据，转成字符串
        public RequestWrapper(HttpServletRequest request) {
            super(request);
            StringBuilder stringBuilder = new StringBuilder();
            BufferedReader bufferedReader = null;
            InputStream inputStream = null;
            try {
                inputStream = request.getInputStream();
                if (inputStream != null) {
                    bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
                    char[] charBuffer = new char[128];
                    int bytesRead = -1;
                    while ((bytesRead = bufferedReader.read(charBuffer)) > 0) {
                        stringBuilder.append(charBuffer, 0, bytesRead);
                    }
                } else {
                    stringBuilder.append("");
                }
            } catch (IOException ex) {

            } finally {
                if (inputStream != null) {
                    try {
                        inputStream.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
                if (bufferedReader != null) {
                    try {
                        bufferedReader.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
            body = stringBuilder.toString();
        }

        @Override
        public ServletInputStream getInputStream() throws IOException {
            final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(body.getBytes());
            ServletInputStream servletInputStream = new ServletInputStream() {
                @Override
                public boolean isFinished() {
                    return false;
                }

                @Override
                public boolean isReady() {
                    return false;
                }

                @Override
                public void setReadListener(ReadListener readListener) {
                }

                @Override
                public int read() throws IOException {
                    return byteArrayInputStream.read();
                }
            };
            return servletInputStream;

        }

        @Override
        public BufferedReader getReader() throws IOException {
            return new BufferedReader(new InputStreamReader(this.getInputStream()));
        }

        public String getBody() {
            return this.body;
        }

        // 赋值给body字段
        public void setBody(String body) {
            this.body = body;
        }
    }

}
