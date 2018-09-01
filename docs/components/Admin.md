### Admin 组件源码分析
```jsx
import React, { createElement } from 'react';
import PropTypes from 'prop-types';
import { createStore, compose, applyMiddleware } from 'redux';
import { Provider } from 'react-redux';
import createHistory from 'history/createHashHistory';
import { Switch, Route } from 'react-router-dom';
import { ConnectedRouter, routerMiddleware } from 'react-router-redux';
import createSagaMiddleware from 'redux-saga';
import { all, fork } from 'redux-saga/effects';
import withContext from 'recompose/withContext';

import { USER_LOGOUT } from './actions/authActions';

import createAppReducer from './reducer';
import { adminSaga } from './sideEffect';
import { TranslationProvider, defaultI18nProvider } from './i18n';
import CoreAdminRouter from './CoreAdminRouter';

const CoreAdmin = ({
    appLayout,
    authProvider,
    children,
    customReducers = {},
    customSagas = [],
    customRoutes = [],
    dashboard,
    history,
    menu, // deprecated, use a custom layout instead
    catchAll,
    dataProvider,
    i18nProvider = defaultI18nProvider,
    theme,
    title = 'React Admin',
    loading,
    loginPage,
    logoutButton,
    initialState,
    locale = 'en',
}) => {
    const messages = i18nProvider(locale);
    const appReducer = createAppReducer(customReducers, locale, messages);

    const resettableAppReducer = (state, action) =>
        appReducer(action.type !== USER_LOGOUT ? state : undefined, action);
    const saga = function* rootSaga() {
        yield all(
            [
                adminSaga(dataProvider, authProvider, i18nProvider),
                ...customSagas,
            ].map(fork)
        );
    };
    const sagaMiddleware = createSagaMiddleware();
    const routerHistory = history || createHistory();
    const store = createStore(
        resettableAppReducer,
        initialState,
        compose(
            applyMiddleware(sagaMiddleware, routerMiddleware(routerHistory)),
            typeof window !== 'undefined' && window.devToolsExtension
                ? window.devToolsExtension()
                : f => f
        )
    );
    sagaMiddleware.run(saga);

    const logout = authProvider ? createElement(logoutButton) : null;

    return (
        <Provider store={store}>
            <TranslationProvider>
                <ConnectedRouter history={routerHistory}>
                    <Switch>
                        <Route
                            exact
                            path="/login"
                            render={props =>
                                createElement(loginPage, {
                                    ...props,
                                    title,
                                })
                            }
                        />
                        <Route
                            path="/"
                            render={props => (
                                <CoreAdminRouter
                                    appLayout={appLayout}
                                    catchAll={catchAll}
                                    customRoutes={customRoutes}
                                    dashboard={dashboard}
                                    loading={loading}
                                    loginPage={loginPage}
                                    logout={logout}
                                    menu={menu}
                                    theme={theme}
                                    title={title}
                                    {...props}
                                >
                                    {children}
                                </CoreAdminRouter>
                            )}
                        />
                    </Switch>
                </ConnectedRouter>
            </TranslationProvider>
        </Provider>
    );
};

const componentPropType = PropTypes.oneOfType([
    PropTypes.func,
    PropTypes.string,
]);

CoreAdmin.propTypes = {
    appLayout: componentPropType,
    authProvider: PropTypes.func,
    children: PropTypes.oneOfType([PropTypes.node, PropTypes.func]),
    catchAll: componentPropType,
    customSagas: PropTypes.array,
    customReducers: PropTypes.object,
    customRoutes: PropTypes.array,
    dashboard: componentPropType,
    dataProvider: PropTypes.func.isRequired,
    history: PropTypes.object,
    i18nProvider: PropTypes.func,
    initialState: PropTypes.object,
    loading: componentPropType,
    locale: PropTypes.string,
    loginPage: componentPropType,
    logoutButton: componentPropType,
    menu: componentPropType,
    theme: PropTypes.object,
    title: PropTypes.node,
};

export default withContext(
    {
        authProvider: PropTypes.func,
    },
    ({ authProvider }) => ({ authProvider })
)(CoreAdmin);
```

通过上面代码，我们知道这是一个函数式组件（Functional Components) ，他接受如下属性：
```jsx
const CoreAdmin = ({
    // 自定义布局
    appLayout,
    // 自定义身份验证策略
    authProvider,
    // 子组件
    children,
    // 自定义 Redux Reducer
    customReducers = {},
    // 自定义 Redux Saga
    customSagas = [],
    // 自定义路由
    customRoutes = [],
    // 仪表盘
    dashboard,
    // 历史记录
    history,
    // 目前已废弃，自定义菜单
    menu, // deprecated, use a custom layout instead
    // 可以用来自定义 Not Found
    catchAll,
    // 唯一必需的属性，它必须是一个返回一个promise的函数
    dataProvider,
    // 国际化，用来做多语言切换
    i18nProvider = defaultI18nProvider,
    // 自定义主题
    theme,
    // 自定义标题，默认是 React Admin
    title = 'React Admin',
    // 资源加载 loading
    loading,
    // 登录页
    loginPage,
    // 注销按钮
    logoutButton,
    // 初始 Redux State
    initialState,
    // 本地化，默认是英文
    locale = 'en',
}) => {
    ...
}
```

![](../images/CoreAdmin.png)

相关文档，可以查看 [Admin](https://marmelab.com/react-admin/Admin.html)

### 本地化处理
```js
const messages = i18nProvider(locale);
```
1. 分析下这个默认的 i18nProvider(defaultI18nProvider)：
```js
import defaultMessages from 'ra-language-english';

export default () => defaultMessages;
```
我们发现它是直接返回一个箭头函数，调用函数直接返回 react-admin 所支持的英文语言包 [ra-language-english] (https://github.com/marmelab/react-admin/tree/master/packages/ra-language-english)。